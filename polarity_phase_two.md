# Polarity, phase two: making lowering total for implicitly open rows

This plan covers the lowering-side work that the polarity checker change
(design.md "Polarity: Output-Position Tag Unions Are Implicitly Open") left
open. The checker semantics are settled and are not changed here: an
extensionless tag union in an output position of an annotation is
implicitly open (a fresh flex extension, instantiated fresh at every use),
the annotation bounds its own definition (`Tag Not In Annotation`), and a
where-method signature is instantiated per body use and closed per
obligation. Phase two makes postcheck accept every program the checker
accepts, with no panic reachable from a type-correct program.

The bar is production quality, not a prototype. Every behavior change below
is stated as a rule, is declared in design.md (Rewrite Inventory or the
architecture text it amends), and is pinned by tests at each level it
touches: checker integration tests, Monotype/LIR tests, and CLI or platform
fixtures that run the built compiler. Each work item is one jj change,
described before the work starts, with no debug prints and no undeclared
solver mutation.

## 1. Where things stand

Stack on `main` (`96c3b2fa`), bottom to top:

| Change | Title | Scope |
|---|---|---|
| `kusqzzsn` | Refill `any_negative` in the ARC certifier's recycled state | One line in `src/lir/arc_certify.zig`; an upstream merge artifact that stops every `lir`-dependent build on this base. Not polarity. Dropped at the next rebase once `main` carries the fix. |
| `kupupkyt` | Open output-position tag unions implicitly (polarity) | Checker, types, display, tests, design.md |
| `wyzrkrmn` | Drop redundant `..` from output positions in Builtin and fixtures | Mechanical, snapshots |
| `wlylqvxq` | Instantiate where-method signatures per body use | Checker, instantiator |
| `rzysvtry` | Suggest the listed tag a Tag Not In Annotation typo resembles | Report hint (`findBestTypoSuggestion` reused; listed tags captured at mint time, including through alias markers) |
| `rprvoylp` | Close implicitly open tag rows before structural derivation | `closeTagRowsForDerivation` at all seven derivation sites, `RedirectRule.derivation_marker_ext_closure`, design.md Rewrite Inventory entries, six pinning tests |

Verified state at the top of the stack: checker integration suite green
(601/601); Parser CLI suite green (59/59); full `run-test-zig` was
5004/5014 before the last two commits, and those two commits change no
solver behavior outside derivation. The remaining failures are all in
postcheck and are the subject of this plan:

| Symptom | Where | Item |
|---|---|---|
| `resolved Monotype view requested for an unresolved instantiation node` building `test/http-headers/app.roc`, `test/json-decoder/camel_app.roc`, `test/json-decoder/camel_direct_app.roc`, and `roc test` on `test/cli/ParserTopLevelStored*.roc` and `issue_10888_json_parse_repeated_nested_field.roc` | `lower.zig` `resolvedPreparedCodecCallsForBoundary` | W2 |
| `lir_inline_test` "nested iterator results retain the callee-authored representation", case `closed direct Try method` | `checked_artifact.zig` plan classification | W3 |
| `lir_inline_test` "issue 10121 shared JSON helpers preserve optional nested round trips": checker rejects with four `missing method` (encoder) errors — harness path only; the same source checks clean and passes `roc test` on the same-era binary | `Check.zig` encoder derivation, or the harness's import-view topology | W4 |
| Same test after W4: `instantiation widened a closed tag union` during a Builtin lambda specialization — harness path only; the same body lowers and runs on both backends via the CLI | `solve.zig` `unifyTagRows` via `selectExprRepresentationAtNode` | W5 |
| A where-method use that widens its copy panics when the implementation's own return row is closed (`instantiation widened a closed tag union` in `instantiateTargetFromPlanNode`) | `lower.zig` dispatch lowering | W6 |

## 2. Work items

Each item states the decision, the rule as it will be written in
design.md, the code touch points, the tests, the verification, and the
risks. Items are ordered so that every commit compiles and its own tests
pass on top of the previous one.

### W1. Build fix on this base (landed: `kusqzzsn`)

The one-line `refilled_fields` addition sits as the first commit of the
stack so every later commit builds. It is not part of polarity and must not
be squashed into any polarity commit. When upstream fixes the artifact,
rebase and drop it.

### W2. Stored-codec restore: ground row defaults now (W2a), then emit in Phase B (W2b)

**Mechanism.** A stored parser or encoder constant (`parse_headers =
Encoding.HttpHeader.parser_for()`) is restored at its use site by
`restoreConstParserRuntimeFnAtNode` (`lower.zig` ~33872). Restoration
prepares one generated call per format method (`prepareStructuralCodecCallsAtNode`
→ `prepareCustomCodecCallsAtNode` → `prepare*CodecCall`, ~43690–45822). Each
prepare function instantiates the format method's checked scheme fresh
(`target_ctx.instNode(lookup.target.callable_ty)`) and relates only the
encoding, the state, the error row, and the ok row when the result is the
state. The ok-payload protocol union of methods such as
`parse_record_start : … -> Try([Counted(..), Uncounted(..)], [BadHeader])`
is never related to anything. Under polarity that union's extension is a
quantified flex in the Builtin scheme, so the instantiated node is
`InstVariable{ origin = checked_variable, row_default = .empty_tag_union }`,
unresolved. The eager restore then demands resolved views of every prepared
call (`resolvedPreparedCodecCallsForBoundary`, non-frozen branch, ~43821)
before the graph freezes, and Phase-B sealing, which is the one place that
applies row defaults (`GraphTypeFinals` / `materializeUnresolved`), has not
run. Before polarity these positions were closed structures.

**Decision (W2a, transitional).** Ground row defaults at exactly that chokepoint: in
`resolvedPreparedCodecCallsForBoundary`'s non-frozen branch, before
`currentPhaseTypeForNode`, walk `prepared.callable_node` and
`prepared.shape_node` and set every reachable unresolved cell that carries a
`row_default` to the content sealing would give it. Rows only: numeric
default phases stay with `materializeLiteralDefault`'s runtime-demand rule.
Cells without a default are left untouched so the resolved-view invariant
still fails loudly for genuinely unresolved types.

Why this point and not earlier: the prepare functions relate the callee's
error row to the outer result's error row after instantiation. Grounding
per prepared call right after `instNode` would close the callee's error row
before that relation and turn one panic into `instantiation widened a closed
tag union`. The chokepoint runs after every preparation relation and is the
exact mirror of `sealedPreparedCodecCallsForBoundary`, which applies the
same defaults through the sealer in Phase B.

Why W2a is not the end state: design.md already claims the two-phase
discipline for codec generation ("Parser generation runs after the
instantiation graph freezes, so derived-codec parsers obey the Phase-A/
Phase-B boundary", ~5737–5748) and states that a defaultable checked
variable becomes durable `[]` only at final sealing (~7002–7006). The eager
stored-codec restore violates both today; polarity merely made a row reach
it unresolved. The deferred structural path already has the machinery the
restore lacks — a `.pending_deferred` reservation, a boundary record
(`deferred_structural_serializations`), Phase-A preparation of codec calls
and `??` field defaults, and Phase-B emission from sealed types
(`emitDraftDeferredStructuralSerializations`, `sealedPreparedCodecCallsForBoundary`,
`sealedPreparedFieldDefaultsForBoundary`) with the assertion that emission
creates no new runtime demand. W2a is an eager consumer committing a
default, exactly the class the doc comment at `lower.zig` ~15535 forbids
("never by an eager consumer"), and it is the same helper the old branch
grew to thirteen call sites. It lands first because it is thirty lines and
unblocks nine programs; W2b removes it in this plan, not in a follow-up, so
the invariant is restored rather than annotated.

**Rule text for W2a (design.md, the Monotype "defaults apply only at final
sealing" statement ~7005, and the doc comment at `lower.zig` ~15535).** Add
the single declared exception, marked transitional and citing W2b: "A stored codec
restore prepares its generated format-method calls before the graph
freezes and must emit their bodies from resolved views. Immediately before
those views are taken, `InstGraph.groundRowDefaults` commits the row
defaults of every cell reachable from a prepared call's callable and shape
nodes to the content final sealing would materialize. Numeric defaults are
never committed there. A derived codec determines each protocol row exactly,
so no later relation can widen a grounded row; the `unifyTagRows` invariant
enforces that."

**Code.**
- `src/postcheck/monotype/solve.zig`: `pub fn groundRowDefaults(self: *InstGraph, root: NodeId)`, modeled on the old branch's `groundUnresolvedDefaults` but without the numeric arm; `requireRelationProduction()`; visits `list/box/tuple/func/tag_union/record/named` children; `.redirect => unreachable`.
- `src/postcheck/monotype/lower.zig` `resolvedPreparedCodecCallsForBoundary`: in the `!frozen_sealed_emission` branch, `try self.graph.groundRowDefaults(prepared.shape_node); try self.graph.groundRowDefaults(prepared.callable_node);` before the two `currentPhaseTypeForNode` calls, with a comment citing the rule.
- Doc comment updates at `lower.zig` ~15535 and design.md ~7005; extend the Polarity section's lowering note (design.md ~4433). Not the Rewrite Inventory: that inventory classifies solver-mutating rewrites in checking, and `groundRowDefaults` is a Monotype graph mutation.

**Tests.**
- `solve.zig` unit test next to the existing row-default tests (~6290–6420): a func node whose ret is a tag union with an `InstVariable.checkedVariable(null, .empty_tag_union)` tail becomes resolved after grounding; a bare `checked_variable` without default stays unresolved; a numeric-phase leaf stays unresolved.
- CLI: the six registered fixtures (`parallel_cli_runner.zig` ~1509–1515, suite `subcommands`; their names contain "stored", only two contain "stored top-level parser") must pass. `issue_10888_json_parse_repeated_nested_field` is already registered (~1461, "issue 10888: JSON parser retains metadata …", asserting no "postcheck invariant violated").
- Platform: `run-test-zig-http-header-decoder-platform`, `run-test-zig-json-decoder-platform` (all three apps).

**Verification.** `timeout 120 ./zig-out/bin/roc test --no-cache test/cli/ParserTopLevelStoredParser.roc` and siblings; `zig build run-test-cli -- --suite subcommands --filter stored --filter "issue 10888"`; the two platform steps; `zig build run-test-zig-lir-inline` (must stay green: the deferred structural path shares the prepare functions).

**Risks.** (1) The callee spec was keyed as an open request (`draftOpenRequestKey`) before grounding; a later identical closed request may specialize the same format method twice. Not a correctness issue; measure spec counts on the JSON fixtures; W2b removes the cause. (2) `shape_node` grounding is a superset of what is observed unresolved; it is harmless for derived shapes because derivation determines rows exactly, but the unit test must cover a shape with a record whose field carries a defaultable tail. (3) The doc invariant is weakened by exactly one declared case; the Phase-B assertions ("deferred structural serialization changed its sealed result type") stay satisfied because grounding yields the same content sealing would. (4) A format-method implementation whose own protocol row was closed by its body (a closed-source return) meets a grounded request that lists more tags: that is the W6b family inside codec land and W6b's adapter covers it; W2a must not paper over it with a wider grounding.

**W2b. Two-phase stored-codec restore.** Decision: the four BodyContext-level
restores with dead `frozen_sealed_emission` branches (`lower.zig` ~33786,
~33937, ~34104, ~34274: parser and encoder runtime functions, `AtNode` and
plain) split into a Phase-A half that runs while the graph accepts relations
— instantiate the constructor plan against the request, bind const source
captures, restore the encoding capture, `prepareStructuralCodecCallsAtNode`,
prepare `??` field defaults the way `prepareDraftDeferredExprs` does,
`buildParserRestoredPrecomputedPlan`, reserve the runtime boundary
`.pending_deferred`, and append a boundary record alongside
`deferred_structural_serializations` — and a Phase-B half run by the same
pass as `emitDraftDeferredStructuralSerializations`: sealed prepared calls
and field defaults, `lowerParseResultFromState` against sealed `TypeId`s,
`addFn` with a `.sealed` `mono_fn_ty`, the capture lets, and
`fillExprReservation`. The eager `sameClass(parsed_node, runtime_fn.ret)`
check becomes the Phase-B `typeEql` assertion. The non-frozen branch of
`resolvedPreparedCodecCallsForBoundary` and `groundRowDefaults` are deleted.
The Builder-level `restoreConstParserRuntimeFnExpr` (own graph, sealed by
`sealActiveBodyDraft`, no prepared codec calls) is unaffected. Rule text:
delete W2a's exception; the ~7005 statement and the ~15535 doc comment read
as on `main`, and the Phase-A/Phase-B paragraph at ~5737 gains one sentence:
"Stored codec restores (`parser_runtime` / `encoder_for_runtime` constants)
prepare in Phase A and emit in Phase B like every other codec body." Tests:
the same fixtures; `run-check-snapshots` on the JSON and http-headers
fixtures must show no lowered-output change against W2a (the sealed body
equals the eagerly emitted one). Cost: medium — the mechanism exists, the
work is moving the emission halves across the freeze and making the stored
path prepare field defaults; I could not size it more precisely without a
build. Risk: `enterCallableBodyDemandScope` and `constFnEvidence` must be
valid in Phase B; the "produced a new checked runtime-value demand"
assertion is the guard.

### W3. Plan classification: a body-local defaultable row tail is closed

**Mechanism.** `specializeResolvedStaticDispatchPlanCallables`
(`checked_artifact.zig` ~21644) decides `direct_closed` vs
`direct_parametric` with `rootContainsIdentityVariables(plan.callable_ty)`
(~21684, and the iterator twin ~21713). Publication marks every flex as an
identity variable (~7791), so a method whose return row is
`Try(Iter(U64), [Unavailable])` with a defaultable flex tail is now
`direct_parametric`. Monotype's parametric path bails at
`completeDeferredIteratorResult` (`lower.zig` ~32164) because the request
node is not resolved, the callee's private iterator representation is never
adopted, and `Builtin.Iter.next` becomes reachable. The closed path already
seals such tails (`lowerCheckedTypeVariable`, ~6348, `row_default →
empty tag union`).

**Decision.** The classification keeps the meaning design.md gives it
("independent of the enclosing specialization", ~7756), which is not the
same as "carries a row default". A defaultable, unconstrained flex row tail
is independent only when no specialization edge can bind it: it is not an
identity variable of the enclosing template's checked function root, of
that root's where-clause signatures, or of a nested generalized scope's
root. The failing iterator case qualifies (`wrapped = rows.wrapped()` is
body-local). A tail shared with the enclosing function's own return row
does not, and the earlier draft's rule ("defaultable ⇒ closed") breaks it:

    Rows := {}.{ wrapped : Rows -> Try(Str, [Unavailable]), wrapped = |_| Ok("x") }
    wrap : Rows -> Try(Str, [Unavailable])
    wrap = |rows| rows.wrapped()
    use : Rows -> Try(Str, [Unavailable, Other])
    use = |rows| { s = wrap(rows)?  Ok(s) }

(`OpenMethodWidenedCaller.roc`, passes today on the built compiler on both
backends because the plan is `direct_parametric`.) The dispatch's callable
ret tail is `wrap`'s implicitly open extension, which `use`'s `?` widens
to `[Unavailable, Other]` in `wrap`'s specialization. Classified closed,
`lowerClosedDirectProcedureDispatch` (~37879) lowers `plan.callable_ty`
through `lowerType`, sealing the tail to `[]`, then
`constrainTypeToMono(checked_ret_ty, function.ret)` exact-unifies that
closed `Try(Str, [Unavailable])` with the request's `[Unavailable, Other]`:
`instantiation widened a closed tag union`.

Mechanism: a predicate on the checked type store,
`callableIdentityIsSpecializationIndependent(callable, enclosing_identity)`,
walking the payload: `.rigid` → parametric; `.flex` with `row_default ==
null` or `numeric_default_phase != null` → parametric; a defaultable `.flex`
that is a member of `enclosing_identity` → parametric; otherwise closed.
`enclosing_identity` is the identity-variable set of the enclosing template
root plus its where-clause signatures and nested-scope roots; the root
builder already collects identity-variable slots per published root
(`identity_variables`, ~6956/7566), so `specializeResolvedStaticDispatchPlanCallables`
is driven per template over its plan-ref span (or receives the set per
plan) instead of over the flat plan table. `rootContainsIdentityVariables`
is unchanged: its other consumers — the substitution fast path (~4932) and
the payload identity walk that decides digest identity (~6602) — need the
var-based meaning.

Rejected alternative: seal defaultable tails on the Monotype side before
the `typeIsResolved` gate in `completeDeferredIteratorResult` (~32164).
That makes iterator completion a second eager consumer of row defaults
(W2a's class) and leaves every polarity-opened method call
`direct_parametric` — a precision and compile-time regression against
`main`, where the same calls were closed.

**Rule text (design.md "Static Dispatch In Monotype", the `direct_closed`
bullet ~7756).** "A checked flex row tail that carries a row default and no
constraints, and that the enclosing template does not quantify (it is not
an identity variable of the template's root, its where-clause signatures,
or a nested generalized scope), has exactly one instantiation — its row
default — and does not make a direct plan parametric; the closed path seals
it to that default (`lowerCheckedTypeVariable`). A tail the enclosing
template quantifies is parametric, as any other identity variable."

**Tests.** `lir_inline_test` "nested iterator results…" (all seven cases);
`OpenMethodWidenedCaller` as a `lir_inline`/CLI fixture on both backends
(pins the exclusion: must remain `direct_parametric` and pass); the same
program called only at its own row (still parametric by rule; pins that
the rule is about quantification, not about observed widening); one case
with an explicit extra tag and a named extension to pin the rigid side.

**Verification.** `zig build run-test-zig-lir-inline`; the full lir-inline
suite; snapshots.

**Risks.** Any other place that expects `direct_parametric` for defaultable
tails; grep every consumer of the classification before changing it. The
compiler already carries two notions of variable identity — digest identity
(any variable, `rootContainsIdentityVariables`) and compile-time-root
concreteness (`checkedTypeIsConcreteCompileTimeRoot`, `.flex => false`,
~1703) — and boxy and glue already treat a defaultable tail as closed
(`boxy/plan.zig` ~5360, `glue.zig` ~4434). This predicate is a third; the
commit documents all three side by side so they cannot drift silently. The
old branch's sprawl is not re-imported: the compile-time-root gate and the
digest dedup (`vkoyrzmonxko`, `tzskosxk`) stay untouched, so no constant
moves between the stored and eval paths.

### W4. Encoder derivation tolerates implicitly open optional rows

**Mechanism.** `Shape : { item : Try({ bar : Str, count : U64 }, [Missing]) }`
puts `[Missing]` behind an alias marker that resolves to a fresh flex at each
use. The parser derivation already tolerates `[Missing | flex]`
(`unboundTryInfoFromNominal`, pinned by `pinWildcardOptionalParseField`),
but the encoder derivation requires a closed row
(`varSupportsDerivedEncodeRecordField` → `missingTryInfoFromNominal` →
`varIsExactUnitTagUnion` → `tagExtIsClosedEmpty`) and reports `missing
method` for `encoder_for`. `[Null]` has the same requirement on both sides.

**Reproduction (review finding).** The exact 10121 source as a CLI module
(`Issue10121Exact.roc`, `main : Bool` evaluated at compile time) and the
same body as a runtime function (`Issue10121Fn.roc`) check with 0 errors
and pass `roc test --no-cache` on both backends with the same-era binary
the report used (built before `rprvoylp`). The four `missing method` errors
are reachable only through the `lir_inline_test` harness
(`compileInspectedProgramForTargetWithBuiltin(.module, …, .native)`). The
harness's import-view topology is the first suspect:
`lookupMethodTargetAcrossViews` (~17458) documents that snapshot-style
compiles pass Builtin only as a direct import and miss builtin-owned
methods when `available` alone is searched, which is exactly the shape of
"missing method (encoder_for)". W4's first step is to name the harness
difference (import views, executable-root checking mode, target) and decide
whether it is a harness bug or a real checker path the CLI does not take;
the fix goes where the difference is, and the harness stays the gate.

**Decision.** `rprvoylp` closes every reachable flex extension before the
encoder's eligibility check and pins an encoder derivation over
`Try(Str, [Missing])` fields, so if the harness path is a real checker
path this diagnosis is expected to be subsumed. Run `lir_inline_test`
issue 10121 on the stack; it must get past type checking. If it does not
and the cause is the checker, the fix is to
make `closeTagRowsForDerivation` reach the encoder's eligibility site (the
row is still open there because closure ran too late or on the wrong var),
not to add parser-style tolerance (`unboundTryInfoForVar`) to the encoder:
that would be two mechanisms for one declared rule. Either way, add the
`[Null]` encoder and parser cases as checker tests: they share the
closed-row requirement and are not pinned today.

**Tests.** Checker integration tests for encoder derivation of a record with
`Try(_, [Missing])` and `Try(_, [Null])` fields reached through an alias;
`lir_inline_test` issue 10121 must get past type checking.

### W5. Closed row against a widened request in a Builtin body (diagnosis only)

**Mechanism (located; provenance unknown).** With W4 in place, issue
10121 panics with `instantiation widened a closed tag union` from
`selectExprRepresentationAtNode` → `selectRequestRepresentation` inside a
pending spec job for a Builtin lambda. The rows are expected
`[InvalidJson(Str), MissingRequiredField(Str)]` (closed) against produced
`[InvalidJson(Str)]` (closed).

The reports' hypothesis — that `Json.invalid_json : [InvalidJson(Str), ..]`
reaches Monotype sealed closed while the request was widened — is
contradicted by scratch programs run on the built compiler (`roc check` and
`roc test --no-cache`, all green): a same-module `v : [A, ..]` used at
`Try(Str, [A, B])` and at `[A, C]` (so it generalizes: `..` sets
`mentions_type_var`, `isGeneralizableValueBinding` ~20609); `Err(Json.invalid_json)`
used at `Try(Str, [InvalidJson(Str), Other])` from a user module; and a
function parsing the 10121 `Shape` alias at `[InvalidJson(Str),
MissingRequiredField(Str)]`. The reason is structural: a generalized value
is not a concrete compile-time root (`checkedTypeIsConcreteCompileTimeRoot`,
`.flex => false`, ~1703), so it is specialized per request on the eval path
and no stored ground row exists to disagree with; a weak value's row
accumulates every same-module widening before `closeWeakValueImplicitOpenExts`
grounds it (`w : [A]` used at `[A, B]` passes; a second use at `[A, C]` is
rejected by the audit, as designed); and cross-module widening of a weak
value is a checker mismatch. Neither candidate fix of the earlier draft is
supported: (a) the produced-value witness widening is the abandoned
old-branch change and addresses a situation the current checker semantics
cannot produce; (b) assumes a template sealed at its own row, which the eval
path does not do. Both are struck.

The 10121 shape itself is not the cause either: the exact source as a
compile-time value root (`Issue10121Exact.roc`), the same body as a runtime
function on both backends (`Issue10121Fn.roc`), and the intermediate forms
(`MainValueRootBoth.roc`: value root, both codecs over the same alias, a
local annotated `Try` binding; `MainValueRoot.roc`; `LocalAnnotatedInFn.roc`)
all check clean and pass `roc test --no-cache` on the same-era binary. Like
W4, the panic is reachable only through the `lir_inline_test` harness
path. A Builtin that the harness compiles or publishes differently from
the CLI — weak-value grounding or derivation closure not applied to the
harness's Builtin view, or a Builtin helper whose annotated row the
harness's checker closed by a closed-source return (the W6b family) —
would explain a closed one-tag `[InvalidJson(Str)]` row the CLI never
produces. Producers to check on that path: the `MissingRequiredField`
injection (`ensureGraphParserMissingRequiredFieldError` ~45512,
`relateCustomParserErrorInjection` ~43626), which widens the outer result's
error row against a closed request; and the Builtin bodies the report
named. The json-decoder camel failure carries a different message ("unified
a tag union with a non-tag-union type" under `restoreCapturingConstFnAtNode`)
and must not be assumed to be this family until W2 has cleared it.

**Decision.** Diagnose on the harness path, reproduce minimally, then fix
under whichever item's rule owns the producer; no fix is chosen here. (1)
Name what the harness does differently (W4's first step; shared). (2) One
traced run naming the checked expression, the template, and both row
producers at `selectExprRepresentationAtNode`. (3) A minimal `lir_inline_test`
reproduction — the scratch programs above are known-good CLI controls to
diff against. (4) Classify: a closed-source row inside Builtin is W6b's
family and lands after W6b on its mechanism; a codec preparation relation
belongs to W2; a closed row the checker published where polarity says open
is a checker bug and lands before W6, declared in the Polarity section; a
harness-only topology difference is fixed in the harness, with the CLI
controls added as fixtures so the two paths cannot drift again. The fix
commit carries the reproduction that names the producer.

### W6. Where-method uses: deliberate per-use plans and closed-implementation re-tag

**Mechanism (verified with the built compiler).** A body use that widens
its where-method copy already lowers correctly when the implementation's
own return row is open: the artifact's plan carries the use's copy as
`callable_ty`, `paramIndexFor` cannot match the copy's fresh root to the
where-clause var and falls back to a same-name match with
`independent_callable = true`, and Monotype then instantiates the
implementation's scheme against the use's callable. Widening, `?` into a
wider row, exhaustive closing, two independent uses, and nested evidence all
pass on both backends. It panics only when the implementation's return row
is closed in its scheme (the body returns a top-level constant, an
input-position parameter, or a nominal field):
`instantiateTargetFromPlanNode` → `relateFunctionRequestInterface` →
`unifyTagRows` "instantiation widened a closed tag union".

**W6a. Make per-use plans deliberate.** The working behavior rests on a
fallback intended for a different case. Use the scheme-use mechanism that
exists rather than a parallel pair table: a body use is a scheme
instantiation (`instantiateWhereMethodForUse` ~6198 copies the signature's
structure over shared leaves), and `SchemeUseRecord` (`ModuleEnv.zig` ~685)
already models one — a slot kind, a `slot_data` unique per constraint
instantiation (the use's constraint fn var, as `dispatch_target` does), the
scheme root (the where-clause signature var), and copy pairs. Add
`Slot.where_method_use`; record it in `checkStaticDispatchConstraints`'
rigid branch (~29961) when a copy is minted, pairs from the instantiator's
`var_map`; have `paramIndexFor` (~17512) resolve the plan's
`constraint_fn_var` through that record to the signature root before
matching and produce `evidence_dependent{ independent_callable = true }` by
rule; the same-name fallback becomes an invariant. The evidence pass's
walks over `scheme_uses` (`emitSchemeUseSiteEvidence` ~18469,
`evidenceRefsForRecord`) skip the new slot: a body use has no obligations
of its own. Old-branch lessons: keyed by the two explicit vars (use fn var,
signature fn var), never by name; `evidenceNodeForTarget` memoization is
unaffected because per-use copies never reach a `record_idx`. Update the
`independent_callable` doc (`static_dispatch_registry.zig` ~1502), which
today describes only the shared-slot case.

Risk, to be settled in this item: an independent callable's nested evidence
is `.synthesize`, which lowering maps to `.from_callable`
(`appendConstFnEvidence` ~4346). An implementation whose evidence schema is
`requires_record` (`procedureEvidenceSchemaFromSlices` ~17963: a param
sourced from `constraint_callable`, `use_site_only`, `erased_row_remainder`,
or a pathed `explicit_default`) cannot be synthesized from the callable.
W6a either reuses the obligation's `.resolved` vector for target identity
and nested evidence while keeping the callable independent, or pins such
targets unreachable with an invariant and a fixture. Tests: the scratch
programs from the investigation as `lir_inline`/CLI fixtures (widen; widen
with a tag that sorts between `Err`/`Ok` to prove the implementation is
specialized at the wider row; `?` into a wider row; one closing use and one
widening use; an implementation with its own where-clause, exercising
`.synthesize` nested evidence; and an implementation with a
`requires_record` schema, constructed from one of the sources above).

**W6b. Closed implementation, widened use.** Decision: a result-row
widening ADAPTER at the template boundary, generalizing the hosted `Try`
adapter, not a second re-tag site at the dispatch call. The compiler has
this mechanism end to end already: a request wider than a template's
declared closed result row is related component-wise without unifying rows
(`relateHostedTryWidening` ~1432), the template is specialized at its
declared row, and a generated `.checked_generated` adapter at the requested
row calls it and re-tags (`completeTemplateReservation` `.hosted` arm
~4640–4700, `hostedTryAdapterSourceType` ~10843, `hostedTryAdapterBody` /
`hostedTryReturnInjectionExpr` / `errorRowInjectionExpr` ~10941–11060).
Hosted is the instance where the declared row is the host ABI; a Roc
implementation whose published result row is closed (its body returns a
closed-source value) is the other instance, and only where-method uses can
reach it — a direct caller of such a function at a wider row is already a
checker mismatch. Work: (1) lift the declared-vs-requested comparison out
of the `.hosted` arm into a pre-step that also runs for `.roc` templates
whose checked root has a closed result row (bare union or `Try`), taking
the narrowed source type from the REQUEST's tags by the declared labels as
`hostedTryAdapterSourceType` does (a polymorphic implementation's rigid
payloads come from the request, never from `lowerType` of the checked
root); (2) make the relation in `instantiateTargetFromPlanNode` /
`methodTargetNodeFromPlan` (~30540, ~39625) width-aware — arguments exact,
result at included width when the implementation's row is closed and the
plan's row includes it — so the request reaches template completion at the
wider row instead of panicking in `unifyTagRows`; (3) compute the `Try`
capability from the type (`hostedTryAdapterCapabilityForRoot` ~19539 is
already generic over any function returning `Builtin.Try` with a closed
error row) and publish it for every template with a closed result row, not
only behind `isHostedProcedureExpr` (~19758). Chosen because one keyed
mechanism — an adapter per (template, requested type) — serves dispatch
plans, `.synthesize` targets, and iterator plans without touching each
call-lowering path, and the hosted path stops being a special case. Cost:
the hosted arm is restructured (pinned by the existing hosted `?`
fixtures), and the request relation gains a width mode.

Nested positions. Monotype lowering has no user-facing diagnostic channel:
`Common.invariant` is a debug panic that compiles to `unreachable` in
release, `Common.compilerBug` panics in every mode, and nothing under
`src/postcheck/monotype` appends problems. "A build error from lowering"
would be a new mechanism; this plan does not add one. A widened row in a
position the adapter cannot re-tag (inside a `List`, a record field, a
tuple, a tag payload, a non-`Try` nominal) is decided by the checker. The
earlier draft's claim that check-time rejection is not expressible is wrong
in the direction that matters: a body use's widening is observable when
the constrained function's body is checked (its fresh extension resolved
to a row carrying tags — the audit's own test), and each marker's position
in the signature is known when it is minted. Two checker shapes are
possible; Jared decides (open question 3): (d) per-use opening is
restricted to the positions the adapter can re-tag — the direct result row
and a `Try`'s rows — and every other output position of a where-method
signature stays closed as written, so a nested widening is an ordinary
mismatch at the body use and the set of opened positions grows with the
coercion generator; or (e) per-use opening stays everywhere and the
obligation reports a new problem when the implementation's row at a
widened nested marker is closed, which needs the widened markers recorded
per signature and the implementation's scheme inspected before the
obligation unifies it. Until decided, the nested-position fixture asserts
a checker rejection, not lowering behaviour.

Rule text (design.md, a new "Result-Row Widening Adapter" section beside
Hosted Try Question Widening, which becomes its first instance; the
where-method paragraph's lowering note cites it): "A procedure template
whose published result row is closed may be requested at a row that
includes it — the same tags with usable payloads, plus others — when a
where-method body use widened its copy of the signature and the obligation
resolved to that implementation. The request is related component-wise
without unifying the rows, the template is specialized at its declared
row, and a generated adapter at the requested row calls it and re-tags the
result. Only the direct result row and a `Try`'s rows are adapted; a
hosted template is the instance where the declared row is the host ABI."
Tests: `WidenClosedImpl`, `WidenParamImpl`, `QuestionClosedImpl` on both
backends; a closed implementation reached through `.synthesize` nested
evidence; a closed implementation with a rigid payload, pinning the
request-derived narrowing; the hosted `?` fixtures unchanged; the
nested-position fixture per the checker decision.

**Docs.** Rewrite the "Lowering note" in design.md's where-method paragraph
(~4467): open implementations specialize per use as a plain scheme
instantiation; closed implementations get a widening adapter.

### W7. Documentation and description

Update design.md's Polarity lowering note to describe W3 and W6 as declared
rules and to state that stored-codec restores are Phase-A/Phase-B consumers
(W2b). Nothing in this plan is a Rewrite Inventory entry: that inventory
classifies solver-mutating rewrites in checking, and `groundRowDefaults`
(deleted by W2b) and the adapter are Monotype mechanisms declared in the
Monotype sections. If W6b's nested-position decision adds a checker
rejection, that rule is declared in the Polarity section. `kupupkyt`
already bumped the checked-artifact cache version (`CACHE_VERSION` 72 → 73,
`src/compile/cache_config.zig`) because published `row_default`s and
weak-value grounding changed the artifact's meaning; W6a's new
`SchemeUseRecord` slot kind and any W3 checker-published flag change the
artifact again and bump it once more. Refresh the PR description's
verification section.

## 3. Sequencing and commit stack

Order: W1 (already present, move to the bottom) → W2a → W3 → W4 → W5
(diagnosis) → W6a → W6b → W5 (fix, on whichever item's rule owns it) →
W2b → W7. W2a through W4 are independent of each other and can be developed
in parallel worktrees but land in this order so each commit's verification
is monotone. W5's diagnosis depends on W4; its fix may depend on W6b. W6b
depends on W6a's fixtures. W2b is independent and lands last among the
code items so the nine unblocked programs are green early; it is not
optional (open question 1). Every commit is created with `jj new -m` before
its first edit, carries the trailer lines, and is verified in isolation
with its item's commands before the next starts; the full `run-test-zig`
and `run-check-snapshots` run after W3, after W6b, and after W2b.

## 4. Verification matrix

| Level | Command | Gate |
|---|---|---|
| Checker | `zig build run-test-zig -- --test-filter "check type"` | green after W4, W6a, and W6b's nested-position rule |
| Monotype/LIR | `zig build run-test-zig-lir-inline` | green after W3 (iterator, `OpenMethodWidenedCaller` stays green), W6, W5's fix (10121) |
| Stored codecs | `zig build run-test-cli -- --suite subcommands --filter stored --filter "issue 10888"` | green after W2a; unchanged lowered output after W2b |
| Where-method fixtures | `roc test --no-cache --opt=interpreter` and `--opt=dev` on the W6 fixtures | green after W6a (open impls), W6b (closed impls) |
| Platforms | `zig build run-test-zig-http-header-decoder-platform`, `zig build run-test-zig-json-decoder-platform` | green after W2a; camel variants may need W5's fix — verify rather than assume |
| Everything | `zig build run-test-zig`, `zig build run-check-snapshots` | 100% after W6b and again after W2b |

## 5. Risks and rollback

- W2a weakens one documented invariant by a declared, transitional
  exception; if a later relation ever widens a grounded protocol row, the
  `unifyTagRows` invariant surfaces it at the exact site. Rollback is
  removing the two calls. W2b restores the invariant; its risk is the
  size of the emission move, guarded by the Phase-B assertions.
- W3 changes plan classification; every consumer of the classification is
  enumerated in the commit message, and the quantified-tail exclusion is
  pinned by `OpenMethodWidenedCaller`. Rollback is restoring the two call
  sites.
- W6b restructures the hosted adapter arm into a general one; the hosted
  `?` fixtures are the regression gate. The adapter re-tags only the
  direct result row and a `Try`'s rows; every other position is a checker
  decision, so lowering cannot silently produce a wrong representation and
  never reports.
- W5 has no fix chosen; if its diagnosis names a checker-published closed
  row, the fix is a checker change that must be declared before W6 lands.
- The stack sits on a `main` that needs W1 to build; if upstream fixes the
  artifact first, rebase and drop W1.

## 6. Follow-ups, deliberately out of scope

- General row-subsumption coercions (closed values widening into open rows
  in any position), which would also let a closed body publish an open row.
  W6b's adapter is their first instance; nested positions extend it (and,
  under option (d), open the corresponding signature positions) rather
  than rewrite it.
- Cross-module widening of annotated weak values (currently grounded closed
  by `closeWeakValueImplicitOpenExts`, as on `main`). Grounding in the
  checker is the right boundary: a weak value has one representation, its
  row is a module-local shared variable, and importers, glue
  (`glue.zig` ~4434 already reads a defaultable tail as closed), the LSP,
  and the artifact cache all see the type `main` published. Widening it
  across modules would need the same coercion as above, not a lowering
  default.
- Re-keying open-keyed format-method specializations (moot after W2b).

## 7. Open questions for Jared

1. W2b in this PR or the next? The plan says this PR. If it slips, the
   design.md exception for W2a must say "transitional, removed by <change>"
   and W2b is the first follow-up, because `groundRowDefaults` left in the
   tree is the helper the old branch grew to thirteen call sites.
2. W3's `enclosing_identity` set: computed in the artifact per template
   from its plan-ref span (the root builder's identity slots), or published
   by the checker as a per-def flag on the extension var? Both are
   possible; the artifact side keeps the checker unchanged, the checker
   side is cheaper if the plan-ref walk turns out awkward. I could not
   settle this without reading more of the template walk.
3. W6b nested positions: (d) restrict per-use opening to the positions the
   adapter can re-tag, or (e) keep opening everywhere and reject a closed
   implementation at a widened nested marker at the obligation. (d) is one
   small table shared with the coercion generator and keeps every
   rejection an ordinary mismatch; (e) accepts more programs today (open
   implementations at nested positions, which work) at the cost of new
   checker bookkeeping and a new problem kind.
4. W6b mechanism: the adapter at the template boundary is my choice over
   the earlier draft's call-site wrap. If restructuring the hosted arm is
   judged too risky for this PR, the fallback is the call-site wrap with
   the same rule text minus "hosted is an instance" — it is a second
   re-tag site and should then be listed as debt.
5. W5: if the diagnosis shows the checker publishing a closed row where
   polarity says open, is that fix in scope here (it changes checker
   semantics the plan declares settled)?
6. W4/W5 reproduce only through the `lir_inline_test` harness on the
   same-era binary; the CLI accepts and runs the exact 10121 program. If
   the cause is the harness's import-view topology (Builtin passed only as
   a direct import), is fixing the harness acceptable as the W4/W5
   resolution, with the CLI controls promoted to fixtures — or must the
   checker be made to tolerate that topology because other snapshot-style
   consumers (snapshots, the LSP) share it?

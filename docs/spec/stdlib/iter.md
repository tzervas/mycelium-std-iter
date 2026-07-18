# Spec — `std.iter` (iterator / fold / transducer combinators over the kernel `for` fold)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-iter` (M-526, #167, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.iter` · Ring 2 (RFC-0016 §4.2) · Tier B (RFC-0016 §4.4) |
| **Tracks** | M-526 (#167) — the Phase-5 task this spec delivers |
| **Scope** | Iterator / fold / transducer combinators (`map`, `filter`, `fold`, `scan`, `zip`, `take`, …) expressed over the RFC-0007 §4.8 `for` fold — the kernel's bounded linear-recursion primitive. The module owns the *combinator surface*; it does not own a new control form. |
| **Boundary** | Does **not** add a new kernel iteration form — RFC-0007 §4.8 already forbids `while`/`loop`/`break` (the divergence bit, §4.5). Does **not** own the *collections* the combinators range over (persistent Vec/List/Map is `collections`, M-511) nor the element-type conversion of a result (`cmp`/`convert`, M-532). An unbounded source (a generator/stream) is an explicit, separately-typed lazy surface, NOT the default. |
| **Depends on** | RFC-0007 §4.8 (the `for` fold + its `Total`-by-construction elaboration) and §4.5 (the divergence bit / `matured` gate); RFC-0016 §4.1 (the per-op contract C1–C6) and §4.4 (the `iter` row); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, the guarantee lattice). |
| **Grounds on** | The L1 kernel calculus realized in `crates/mycelium-l1` (the `for`-sugar desugaring + the §4.5 totality checker, including the M-320 / r4 maturation); Ring 0 `core` (`Option`/`Result`, the lattice tags). KC-3: adds **no** trusted code. |

---

## 1. Summary

`std.iter` is the ergonomic combinator layer — `map`, `filter`, `fold`, `scan`, `zip`, `take`, `count`,
`any`/`all`, … — over the one iteration primitive the kernel actually has: the RFC-0007 §4.8 `for` fold, a
bounded walk of a linearly-recursive value that the §4.5 totality checker classifies **`Total` with zero
extension** (iteration is bounded *by construction* because a value is finite and acyclic, §4.7). The
**honesty crux** is **totality preservation**: a combinator applied to a `for` fold over a finite value
*stays total and terminating*, and the guarantee matrix (§4) records — per combinator — whether it does.
The module structurally forbids the silent-default of "iteration that might not terminate": there is no
ambient `while`/`loop` to wrap (RFC-0007 §4.8 excludes them), so a combinator that *could* diverge — an
unbounded `unfold`/stream — is **not** offered as a total combinator and is **explicitly typed and named
lazy**, never a silent thunk (C1/C2). The module is Ring 2 and adds no trusted code (KC-3): it consumes the
kernel's `for` elaboration and the Ring-0 value model only.

## 2. Scope & module boundary

- **In scope:** the eager combinator surface over a finite linear-recursive source — transforms (`map`,
  `filter`, `flat_map`/`flatten`, `scan`, `enumerate`), reductions (`fold`, `reduce`, `sum`/`product` via a
  declared monoid, `count`, `any`/`all`, `find`/`position` → `Option`), pair/merge combinators (`zip`,
  `unzip`, `chain`), bounded slicing (`take`, `skip`, `step_by`), and a **transducer** form (a composable,
  source-independent step transformer that fuses into a single `for` fold). An explicitly-named **lazy**
  surface (`lazy_unfold`, `lazy_take`) is in scope only as the honest home for *unbounded* generation.
- **Out of scope (and who owns it):**
  - The **collections** the combinators range over (persistent Vec/List/Map/Set/Deque) and their own
    iteration adapters — `collections` (M-511). `iter` ranges over them; it does not define them.
  - **Element conversion** in `map`/`zip` results (lossy/typed convert) — `cmp`/`convert` (M-532); a `map`
    closure may *call* a convert op, but `iter` does not narrow types itself.
  - **A new kernel iteration form.** RFC-0007 §4.8 owns the only one; `iter` is sugar/library over it, never
    a calculus extension (it cannot reintroduce `while`/`break`, §4.8 "What stays out").
  - **Effectful iteration drivers** (IO over a `substrate`) — `io` (M-514) / the `substrate` affine
    discipline (LR-8). `iter` is value-semantic; an effectful step declares its effect (C6) but the
    consumed-once handle lives in `io`.
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). It **builds new** combinators (it does not merely re-export a
  crate) **over** the kernel `for` elaboration (RFC-0007 §4.8) and Ring-0 `core`. KC-3: no new trusted code —
  every combinator's termination is *inherited* from the kernel's `Total`-by-construction fold, not asserted
  by this module.

## 3. Exported-op surface (design sketch)

A design sketch — enough to fix the surface and feed the §4 matrix, not a committed grammar. Combinators are
value-semantic pure functions (C4); fallible reductions return `Option`/`Result`; the lazy surface is a
*distinct, named type*, never an implicit thunk.

```
// illustrative signatures (not a committed surface)

// --- the substrate: anything walkable by the kernel `for` fold ---
// `Foldable<E>` = a linear-recursive value of element type E (RFC-0007 §4.8 shape:
// nil | cons(E, T)). Eager combinators range over Foldable and lower to ONE `for` fold.
// (Whether `Foldable` is a trait or per-type monomorphic is Q2 — RFC-0007 r4 defers traits.)

// --- transforms (Foldable in, Foldable/value out) — total over a finite source ---
map      : (Foldable<E>, fn(E) -> F)            -> Foldable<F>
filter   : (Foldable<E>, fn(E) -> Bool)         -> Foldable<E>
scan     : (Foldable<E>, A, fn(A, E) -> A)      -> Foldable<A>   // running accumulator, length-preserving
enumerate: (Foldable<E>)                        -> Foldable<(Nat, E)>
flat_map : (Foldable<E>, fn(E) -> Foldable<F>)  -> Foldable<F>   // finite-of-finite stays finite

// --- reductions (Foldable in, value out) — these ARE a `for` fold directly ---
fold     : (Foldable<E>, A, fn(A, E) -> A)      -> A             // the §4.8 primitive, surfaced
reduce   : (Foldable<E>, fn(E, E) -> E)         -> Option<E>     // None on empty (C1, not a sentinel)
count    : (Foldable<E>)                        -> Nat
any/all  : (Foldable<E>, fn(E) -> Bool)         -> Bool          // see Q3: short-circuit is fold-internal
find     : (Foldable<E>, fn(E) -> Bool)         -> Option<E>     // None when absent (C1)
position : (Foldable<E>, fn(E) -> Bool)         -> Option<Nat>

// --- pair / merge — total; mismatched lengths are RESOLVED EXPLICITLY, never silently ---
zip      : (Foldable<E>, Foldable<F>)           -> Foldable<(E, F)>   // truncates to min length; see Q1
chain    : (Foldable<E>, Foldable<E>)           -> Foldable<E>

// --- bounded slicing — total; the bound is a value, not a promise ---
take     : (Foldable<E>, n: Nat)                -> Foldable<E>
skip     : (Foldable<E>, n: Nat)                -> Foldable<E>
step_by  : (Foldable<E>, k: Nat)                -> Result<Foldable<E>, ZeroStep>  // k = 0 is an error (C1)

// --- transducer: a source-independent step transformer that FUSES into one `for` fold ---
type Transducer<E, F>                                            // composable; `compose` is associative
transduce: (Foldable<E>, Transducer<E, F>, A, fn(A, F) -> A) -> A

// --- the EXPLICIT lazy surface (NOT total — honestly named, never a silent thunk) ---
type Lazy<E>                                                     // demand-driven; carries a `Declared` tag
lazy_unfold : (S, fn(S) -> Option<(E, S)>)      -> Lazy<E>       // may be unbounded → NOT total (see §4)
lazy_take   : (Lazy<E>, n: Nat)                 -> Foldable<E>   // re-bounds a Lazy back to total
```

`take`/`lazy_take` are the bridge: `take` over a finite `Foldable` stays a single bounded `for` fold;
`lazy_take` is the **only** sanctioned way to turn a `Lazy` source back into a total, foldable value — making
the unbounded→bounded transition an explicit call, not an implicit cutoff.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported combinators. Encoded as a checked table (the RFC-0003 §4 template), asserted in tests once
code lands — never prose only. The **totality-preserving?** column is the key column for this module
(RFC-0016 §4.4, "total + terminating where the kernel guarantees it"). "yes (inherited)" means: over a finite
`Foldable` the combinator lowers to / composes a single RFC-0007 §4.8 `for` fold, which the §4.5 checker
classifies `Total` with zero extension — termination is *inherited from the kernel*, not asserted here.

| Op | Guarantee tag | Totality-preserving? | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|---|
| `map` | `Exact` | **yes (inherited)** — one `for` fold over a finite source | total (closure-fallibility is the closure's tag, not `map`'s) | none (closure may declare its own) | n/a |
| `filter` | `Exact` | **yes (inherited)** | total | none | n/a |
| `scan` | `Exact` | **yes (inherited)** — length-preserving fold | total | none | n/a |
| `enumerate` | `Exact` | **yes (inherited)** | total | none | n/a |
| `flat_map` | `Exact` | **yes (inherited)** — finite-of-finite is finite (§4.7) | total | none | n/a |
| `fold` | `Exact` | **yes (the primitive)** — *is* the §4.8 `for` fold | total | none | n/a |
| `reduce` | `Exact` | **yes (inherited)** | `None` on empty input (C1) | none | n/a |
| `count` | `Exact` | **yes (inherited)** | total | none | n/a |
| `any` / `all` | `Exact` | **yes (inherited)** | total | none | yes (short-circuit decision reified — see §4 note + Q3) |
| `find` / `position` | `Exact` | **yes (inherited)** | `None` when no element matches (C1) | none | n/a |
| `zip` | `Exact` | **yes (inherited)** — truncates to `min` length | total (length policy is explicit, see Q1) | none | yes (the truncation point is reportable — Q1) |
| `chain` | `Exact` | **yes (inherited)** — two finite spines | total | none | n/a |
| `take` | `Exact` | **yes (inherited)** — bound is a `Nat` value | total | none | n/a |
| `skip` | `Exact` | **yes (inherited)** | total | none | n/a |
| `step_by` | `Exact` | **yes (inherited)** | `Err(ZeroStep)` when `k = 0` (C1, no silent step-1) | none | n/a |
| `transduce` | `Exact` | **yes (inherited)** — fuses to one `for` fold | total | none | yes (the fused step pipeline is inspectable) |
| `lazy_unfold` | **`Declared`** | **NO — not total** | (none returned; the source may be unbounded) | none | yes (carries an explicit "lazy / may-not-terminate" `Declared` artifact) |
| `lazy_take` | `Exact` | **yes** — re-bounds `Lazy` to a finite `Foldable` | total *given the bound* (the `Nat` bound makes it terminating) | none | yes (records the bound applied) |

**Tag justification (VR-5 — downgrade rather than overclaim).**

- Every eager combinator is `Exact`/**total** because it lowers to, or composes, the RFC-0007 §4.8 `for`
  fold, whose synthesized helper descends structurally on the spine and is classified `Total` with zero
  extension by the §4.5 checker (RFC-0007 §4.8: "iteration is bounded *by construction* … not by programmer
  promise"). The combinator does **not** re-prove totality; it **inherits** it. This inheritance is the
  module's whole honesty claim, and it holds only because the source is a finite, acyclic value (§4.7) — so
  the matrix scopes "yes (inherited)" to a `Foldable` source explicitly.
- `reduce`, `find`, `position` are `Exact` *and* total, but **fallible**: they return `Option` (empty input /
  no match) rather than a sentinel — the never-silent floor (C1). Fallibility and totality are orthogonal
  here: the function is total (it terminates and returns) *and* honestly partial-in-result (it can return
  `None`).
- `step_by` returns `Result` because a zero step is a genuine error, not a clamp-to-1 (C1).
- `lazy_unfold` is the **one honest exception**: it can produce an unbounded sequence, so it is **not total**
  and is tagged **`Declared`** — the unboundedness is *asserted and flagged*, never proven away. It is a
  distinct, named `Lazy<E>` type (no silent thunk). There is no kernel `for` fold it lowers to *as-is*; only
  `lazy_take` (a `Nat` bound) re-establishes a total, foldable value. This is the module's single place where
  totality is *not* preserved, and it is named, typed, and tagged accordingly — exactly the honesty the
  RFC-0016 §4.4 `iter` row demands ("laziness is explicit").
- **`EXPLAIN`-able (C3):** most combinators are not "selecting/converting/approximating" ops, so they are
  `n/a` for EXPLAIN. The ops that *make a visible decision* expose it: `zip` can report *where* it truncated
  (Q1), `any`/`all` can reify the short-circuit witness (Q3), `transduce` exposes its fused step pipeline, and
  every `Lazy` op carries the explicit lazy/`Declared` artifact (so a reader can always see that a value is
  demand-driven and may not terminate).

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2):** the partial-in-result reductions return `Option` (`reduce`/`find`/`position` →
  `None`, never a sentinel); `step_by(0)` is `Err(ZeroStep)`, never a silent step-of-1; `zip`'s
  length-mismatch policy is an explicit, reportable truncation (Q1), never a silent pad/drop without record.
  Crucially, the *termination* guarantee is never silently dropped: a combinator that could diverge is not
  offered as total at all — it is the named `Lazy` surface.
- **C2 — honest per-op tag (VR-5):** every eager combinator is `Exact`/total by **inheriting** the kernel
  fold's `Total` classification (RFC-0007 §4.8/§4.5); `lazy_unfold` is honestly **`Declared`** (not total).
  The matrix downgrades rather than overclaims — there is no `Proven`-by-assertion totality anywhere; total
  rows trace to the checked kernel property, and the one row that cannot is flagged.
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** decision-bearing combinators reify their decision — `zip`'s
  truncation point, `any`/`all`'s short-circuit witness, the transducer's fused step pipeline, and the
  `Lazy`/`Declared` artifact are inspectable, not opaque. Pure structural transforms (`map`/`filter`/…) make
  no user-visible selection and are `n/a`.
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001):** every combinator is a pure function of its
  inputs (source value + closures) — no hidden mutable cursor; the kernel `for` fold is itself pure structural
  recursion over an immutable value (RFC-0007 §4.7). Laziness is **explicit** (the `Lazy` type), so even the
  demand-driven surface is a value with a declared evaluation discipline, not an implicit side-effecting
  iterator. Metadata (the tags) is not identity (ADR-003).
- **C5 — above the small kernel (KC-3):** the module consumes the RFC-0007 §4.8 `for` elaboration and Ring-0
  `core`; it adds **no** trusted code and re-proves nothing the kernel already certifies. No `wild`/FFI
  (ADR-014) — `iter` is pure-safe.
- **C6 — declared, bounded effects (RFC-0014):** `iter` itself is effect-free; a combinator's signature
  declares **none**. An effect arises only inside a user closure (e.g. a `map` over an `io` step), and that
  effect is declared on the *closure's* signature and surfaces on the combinator's instantiated type — `iter`
  never hides it. The bound on an eager combinator is the finite source itself (no EffectBudget needed); the
  `Lazy` surface carries no implicit budget and is therefore explicitly non-total rather than budget-bounded.

## 6. Grounding

- The single iteration primitive and its **`Total`-by-construction** classification: **RFC-0007 §4.8** (the
  `for`-fold elaboration to a synthesized self-recursive helper; "iteration is bounded *by construction* …
  not by programmer promise") and **§4.5** (the divergence bit + `matured` gate; a `matured` definition must
  be `total`). The exclusion of `while`/`loop`/`break` (RFC-0007 §4.8 "What stays out") is *why* the only
  honest home for unbounded generation is the explicitly-named `Lazy` surface.
- The per-op **contract** and the matrix obligation: **RFC-0016 §4.1** (C1–C6) and **§4.5** (the guarantee
  matrix as a checked table); the **`iter` row** at **RFC-0016 §4.4** ("total + terminating where the kernel
  guarantees it; laziness is explicit") and the Ring-2 placement at **§4.2**.
- The value model, the guarantee lattice `Exact ⊐ Proven ⊐ Empirical ⊐ Declared`, and value-semantics:
  **RFC-0001** / **ADR-003**; the never-silent and honest-tag house rules **G2** / **VR-5**; the small-kernel
  invariant **KC-3**; declared/bounded effects **RFC-0014** (C6).
- Realization base (consumed, not enlarged): the L1 calculus in `crates/mycelium-l1` (the `for`-sugar
  desugaring and the §4.5 totality checker, with the M-320 / r4 maturation extending the structural-descent
  classification). KC-3: this module adds no trusted code.

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) `zip` length-mismatch policy.** The sketch truncates to the shorter spine (the total, never-silent
  default) and makes the truncation point reportable (C3). Alternatives — a fallible `zip_exact` (→ `Err` on
  length mismatch) and/or a `zip_longest` (→ `Foldable<(Option<E>, Option<F>)>`) — are both honest but enlarge
  the surface. **Disposition:** propose `zip` (truncate, reported) as the floor plus a fallible `zip_exact`;
  defer `zip_longest`. Ties to RFC-0016 §8-Q3 (ergonomics vs the contract — how much explicitness at the call
  site).
- **(Q2) Does `iter` need a `Foldable` trait, or does it range over `collections` types directly?** The §4.8
  shape restriction (nil | cons(E, T)) is concrete; whether the combinators are generic over a `Foldable`
  abstraction or monomorphic per linear-recursive type depends on the trait/typeclass story, which RFC-0007 r4
  **defers** (traits/LR-2 are NOT ratified). **Disposition: FLAGGED** — the surface is written abstractly over
  "a linear-recursive value"; the genericity mechanism is pending the trait decision. Ties to RFC-0016 §8-Q1
  (the v0 module set / dependency order) and the deferred RFC-0007 trait work.
- **(Q3) Short-circuit (`any`/`all`/`find`) under a `Total` fold.** RFC-0007 §4.8 excludes `break`; early exit
  is named there as "a later, explicit form (fold-to-`Option`), never ambient control flow." So `any`/`all`/
  `find` must be expressed as a fold whose accumulator *carries a done-flag* (it still walks the full spine but
  stops updating) — total, but not short-circuiting in the cost sense. Whether a true early-termination fold
  (a `for` that may stop before the spine end, still `Total`) is a worthwhile kernel extension is a **FLAGGED**
  question for RFC-0007, not decidable in this module. **Disposition:** ship the done-flag fold (honestly
  total, possibly does extra work); record the cost; flag the kernel question. Ties to RFC-0007 §4.8
  ("fold-to-`Option`").
- **(Q4) The `Lazy` surface — in `iter` at all, or its own module?** A demand-driven, possibly-unbounded
  sequence is the one non-total surface here; keeping it in `iter` risks blurring the total/non-total line the
  module exists to draw. **Disposition: FLAGGED** — keep `lazy_*` in `iter` but **type- and name-segregated**
  (a distinct `Lazy<E>` type, a `Declared` tag, no implicit coercion from `Foldable`), revisit if the surface
  grows. Ties to RFC-0016 §8-Q3 (ergonomics vs contract).
- **(Q5) Transducer composition semantics.** The transducer form promises fusion into a *single* `for` fold;
  the associativity of `compose` and the fusion law (transduce ∘ compose = composed transduce) should be
  stated as a **checked property** (a test, RFC-0016 §4.5) once code lands, not asserted as `Proven` here.
  **Disposition: FLAGGED** — record as an `Empirical`/property obligation for the implementing task; do not
  tag the fusion law `Proven` without a checked theorem (VR-5).

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.iter` module spec (M-526, #167; Ring 2, Tier B)
  as the combinator layer over the RFC-0007 §4.8 `for` fold. Scope = eager transforms/reductions/slicing/
  pair-merge + a transducer form, with an explicitly named, type-segregated `Lazy` surface for the *only*
  unbounded case; boundary disclaims a new kernel iteration form, `collections`, and element conversion. The
  honesty crux is **totality preservation**: the load-bearing §4 guarantee matrix (18 rows) records per
  combinator whether it is total, with every eager op `Exact`/total by **inheriting** the kernel fold's
  `Total`-by-construction classification (§4.8/§4.5) — re-proving nothing (KC-3) — and the single non-total op
  (`lazy_unfold`) honestly tagged **`Declared`**, named, and typed lazy (no silent thunk). §4.1 conformance
  (C1–C6) is stated per clause; grounding traces to RFC-0007 §4.8/§4.5, RFC-0016 §4.1/§4.4/§4.5, RFC-0001,
  G2/VR-5/KC-3, RFC-0014. Five questions FLAGGED (zip length policy, the `Foldable` trait pending RFC-0007's
  deferred traits, short-circuit under a total fold, the `Lazy` surface's home, the transducer fusion law),
  tied to RFC-0016 §8 and RFC-0007 §4.8 where they belong. No code; no kernel change (KC-3). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).

- **2026-06-27 — Self-hosted Tier-0 prototype now executes three-way (M-715, rsm S3).** A *distinct artifact* from this Rust-first spec: `lib/std/iter.myc` (RFC-0031 §5 D4 Tier-0) is the self-hosted Mycelium-lang prototype over a cons-list (`List<A> = Nil | Cons(A, List<A>)`), exporting the **first-order** subset `is_empty_l`/`length`/`map`/`filter`/`foldl`/`any`/`all`/`find` — a strict subset of this spec's eager surface (no `scan`/`zip`/`take`/`transduce`/`Lazy`). Its combinators now **execute three-way** (L1-eval ≡ L0-interp ≡ AOT, `crates/mycelium-l1/tests/std_iter.rs`) after the recursive-HOF defunctionalization re-pass closed (M-715, RFC-0024 §4 extended): a HOF parameter re-passed at a recursive call site (`map(rest, f)`) is threaded through as the **same** static specialization. Honest boundary: HOF arguments are single **named** top-level functions (RFC-0024 static defunctionalization); closures, multi-arg arrows, and a true binary `foldl` (`f: A -> B -> B` combining the accumulator) remain **deferred (M-704)** — the self-hosted `foldl` is the `f: A -> B` discard-acc form, flagged in the nodule. Three-way agreement is `Empirical` (trials); nothing upgraded to `Proven`. This does **not** alter this spec's (Rust-first) surface or status; the full Mycelium-lang migration (M-502-gated) still remains. Append-only; no kernel change (KC-3).

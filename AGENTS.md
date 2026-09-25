# MSO · AI Collaboration Rules

> This file is for AI assistants. When working in a codebase that adopts the Model-Self-Organizing Architecture (MSO), follow this document.
> The human-facing introduction is in [README.md](README.md); this file contains only precise, executable rules. 简体中文版：[AGENTS.zh-CN.md](AGENTS.zh-CN.md)

## Terminology (precise definitions)

| Term | Definition |
|---|---|
| **Model** | Smallest self-contained unit: **owns** its data + carries its rules and operations + exposes only a narrow API. Test: can you state in one sentence what data it owns and what it provides |
| **Aggregate Group** | A natural composition of related models; doubles as the file-organization and code-review boundary. **Not a hard hierarchy** — ownership may be adjusted per circumstances |
| **Interface layer** | Protocol adaptation (HTTP/RPC/CLI/UI views). Business rules forbidden |
| **Business layer** | Use-case orchestration: coordinates models to complete actions. **Must not own rule bodies** |
| **Model layer** | Owner of data and rules. Self-organizing internally: tree composition + lateral calls + cycles allowed with termination guarantees |
| **Application Shell** | Assembly body: wires the three layers into a runnable application (process and entry form, e.g. web service / desktop app / CLI). Contains no business rules; the thinner the better |
| **Composition Root** | The single assembly point: injects ports and assembles shells |
| **port** | The narrow interface a model/aggregate exposes. The only legal channel for lateral calls |
| **Foundation** | Business-agnostic infrastructure (storage/config/logging/utils; termination, budget, re-entrancy-guard & timeout components). Depended on unidirectionally by upper layers |
| **Business-role model** | Answers "who may do what inside a business domain" (e.g. admin/member). An ordinary model inside some aggregate |
| **Shared Model** | Reused by ≥2 aggregates yet **carrying business rules** (unlike the foundation, which has none). Bound by the same three core disciplines |
| **Cross-cutting context** | Public information owned by no model but needed along the whole chain (user identity, tenant, trace ID, locale/timezone, etc.). Propagated implicitly by the foundation, read explicitly; flows without context must declare so |

## Hard Rules

### 1. Dependency direction (one-way; never reversed)

```
Application shell → ① Interface layer → ② Business layer → ③ Model layer → Foundation
```

- Between models: lateral calls allowed, but **only through the other side's port (public operations)**
- Foundation: **must not** import any upper layer
- Interface layer: **must not** contain business rules
- Between model instances across processes/devices: wire protocol only; **within the same process**, ports injected by the composition root may be called directly

### 2. Model-layer core disciplines (only three; process discipline during adoption is in "Incremental Adoption")

1. **Cycles are allowed (A→B→A is legal), but every cycle needs a termination guarantee**:
   - Termination mechanisms come in two layers:
     - **Foundation-provided components**: **budget** (stop when the call budget is exhausted) / **re-entrancy guard** (in-progress marker; re-entry returns immediately) / **timeout**. Implemented and tested once in the foundation; **private implementations in models are forbidden** — home-grown mechanisms are unaware of each other, uneven in quality, and unauditable
     - **Model-layer design option**: **state-machine phase** (each phase only advances forward) — a design constraint, not a sinkable component; must be declared in the model contract
   - Every cycle needs a test proving it stops
   - Language constraint: Go rejects package-level import cycles at compile time. Models start as in-package file groups (naturally legal); when promoting to real packages, models on a cycle merge into one package, or the cycle is cut with an interface wired by the composition root
2. **Narrow API + data ownership**: lateral calls go through the other side's public operations only; data is written by its owner — directly reading or writing other models' internal state is forbidden
3. **No rules hidden in the business layer**: decisions like "who may do X" or "under what condition Y" must live in some model; the business layer only orchestrates

### 3. Granularity is fractal (the same rules hold at every scale)

Model → aggregate → process/microservice are the same idea at different granularities; **the three disciplines hold at every level**:

| Scale | Data ownership | Narrow API | Termination guarantee |
|---|---|---|---|
| Model (in-package) | owns its struct / tables | method interface | re-entrancy guard / call budget |
| Aggregate (file group) | owns its directory / table domain | composition-root-wired port | state-machine phase |
| Microservice (process) | owns its database (no shared DB) | HTTP / gRPC / messaging | timeout / retry budget / circuit breaker |

When reviewing cross-service code, treat "a microservice" as one big model and apply the same checklist: does it own its data? Is its interface narrow? Do cross-service cycles have termination guarantees?

The table lists which mechanisms fit each scale, not who implements them: re-entrancy guards and call budgets are foundation components; the state-machine phase is a design option declared in the model contract.

### 4. Cross-cutting context propagation

**Cross-cutting context**: public information owned by no model but needed along the whole chain — user identity, tenant, trace ID, locale/timezone, etc.

It has no owning model, and the existing disciplines leave it nowhere to go: a shared global variable violates **data ownership** (models co-write one public state); threading it through every method parameter violates the **narrow API** (N layers of calls, N polluted signatures). The only right path:

1. **Implicit propagation**: the foundation/framework carries the context along the execution flow automatically; business code never handles it
2. **Explicit read**: models read values through a foundation-provided uniform read API — no globals, no signature changes
3. **Missing must be explicit**: execution flows without context (scheduled jobs, batch processing, internal triggers) must explicitly declare "this flow has no context", and code must handle the "not found" branch. **Never treat "not found" as "no restriction" and carry on**

Scenario: a request passes through Order → Inventory → Notification. If "current user" travels as a parameter, every method in all three layers needs a userId; as a global, three models co-write one state. The right path: the framework carries it down automatically, each model reads via the uniform API. Anti-example: an internal flow cannot read the "visible scope" context, and the code treats "not found" as "no restriction", exporting data it should not export — missing context silently became default-allow.

### 5. Machine checks (CI blocks; review backstops)

Disciplines enforced only by human review erode — written rules get bypassed and the breach surfaces only after an incident. The machine-checkable subset of the core disciplines must be backed by static checks blocking in CI:

| Discipline | Machine-checkable subset | Not machine-checkable (review checklist backstops) |
|---|---|---|
| Dependency direction | Static import-direction check: lower/same level must not import upper layers or internal files | Whether a call is semantically out of bounds |
| Narrow API + data ownership | A model's public interface signatures must not reference other models' internal types | Whether the data write path is unique |
| Every cycle terminates | Cycle detection + calls on the cycle must hit a foundation termination component or a declared phase mechanism | Business plausibility of "stops" tests |

Good: the Order model imports only the Inventory model's public interface file → check passes. Bad: the Order model imports the Inventory model's internal data file → blocked at commit. Without the check: the violation sits in the code until, three months later, a production incident of models mutating each other's data surfaces the rule nobody followed.

Machine checks are necessary but not sufficient: they catch structural violations, not semantic overreach — the review checklist backstops the latter. Items already blocked by CI need not appear on the review checklist.

### 6. Shared-model admission and destination

The foundation must stay business-agnostic, yet in reality rule-carrying capabilities grow out of business models. Decide **when the second consumer appears**:

- The capability still lives inside the first model and a second model needs it → the latter must not reach into the former; it must be moved out
- Destination: **carries business rules** → solidify into a shared model (bound by the three core disciplines); **pure mechanics, no rules** → sink into the foundation

Scenario: the Order model contains a "spend 100, save 20" discount calculation; the refund flow needs the same calculation — the second consumer has appeared, so the code must move out of the Order model. It carries rules ("who gets which discount"), so it becomes a shared model. A rounding helper carries no business rules — it goes to the foundation.

### 7. Anti-patterns (report on sight during review)

- **God object**: fields and methods of multiple domains share one struct/class; multiple locks share one host
- **Global god-model**: all domains share one model package; a single change ripples across the repo
- **Rules hidden in the business layer**: orchestration code containing "check role/permission/state" decision logic
- **Models poking each other's data**: directly reading or writing other models' internal fields/tables
- **Cycle without termination**: A→B→A with no stopping mechanism
- **Private termination implementations**: models rolling their own budget counters/retry loops instead of using foundation components. Anti-example: many models each implement retries without backoff; the moment the downstream wobbles they all fire at once, and the retry storm takes the downstream down — foundation components ship with backoff and circuit breaking; home-grown wheels have none
- **Context threaded through parameters**: adding a parameter to every layer's methods just to pass cross-cutting context, polluting signatures (right path: "Cross-cutting context propagation")
- **Dedicated big-bang refactor of legacy code**: launching a special project just to "make old code fit the architecture" (right path: "Incremental Adoption" process discipline)
- **A thickening shell**: business logic growing inside the application shell
- **Splitting for its own sake**: a "model" that cannot pass the one-sentence test

## Review Checklist (self-check when writing or reviewing code)

- [ ] Does the new logic have an owner? Which model? Passes the one-sentence test? If not → design the ownership first
- [ ] Do decision rules live in models? Is the business layer only orchestrating?
- [ ] Do cross-model calls go through the other side's public API? Any direct internal-state access?
- [ ] Any new cycle? What is its termination mechanism? Is there a test proving it stops?
- [ ] Did rules leak into the interface layer? Did the shell get thicker?
- [ ] Any dynamic dependency (reflection, string-built import paths) bypassing the static check, breaking dependency direction or reaching into foundation internals?
- [ ] Does each model's file-header contract state: what data it owns / what it provides / **whom it may call** (lateral-call list)?
- [ ] Do shared models (reused across aggregates) still satisfy data ownership + narrow API + termination guarantee?
- [ ] Is the termination guarantee a foundation component (or a declared phase mechanism)? No private budget/retry implementations?
- [ ] Does cross-cutting context travel via foundation implicit propagation + explicit read API? No per-parameter threading? Is the "not found" branch handled?
- [ ] Does a capability gaining a second consumer go through the admission rule (rules → shared model; pure mechanics → foundation)?

## Incremental Adoption (legacy systems · behavior-freezing refactor; full test suite green before each next step)

| Step | Action | Boundary |
|---|---|---|
| 0 Inventory | **Four lenses** to exhaust ownerless things: data ownership / rule invariants / life cycles / use-case flows; produce a 100% file→model mapping | Unassignable files go to a "pending" column with a reason |
| 1 Document | Write a contract per model (owns / provides / allowed-call list) | No structural code changes |
| 2 In-package file groups | Group files by model; write file-header contracts | No new packages, no cross-package file moves |
| 3 Promote to real packages | Promote file groups to independent packages | Models on a cycle merge into one package or cut the cycle with an interface; independent commits per aggregate |

Process discipline:
- Guard against over-splitting: **measure first, then split** — line counts and responsibilities decide; record reasons for anything left unsplit
- Unstructured legacy (no layering, large flat files): contracts first; group files only when a requirement naturally touches them — never launch a dedicated refactor just to "fit the architecture"
- Test files travel with their source in big file reorganizations; run the full gate (vet/test/fmt/frontend tests) at every step
- The model inventory is a living document: new models go through the four-lens check to prevent omissions

## Adoption & Provenance

This section is the adoption procedure. When a project adopts MSO, execute it once, in order:

1. **Vendor the rules**: copy the rules file matching the project's language (`AGENTS.md` for English projects, `AGENTS.zh-CN.md` for Chinese ones) into the adopting project as `docs/mso/AGENTS.md`
2. **Reference the local copy**: record the block below at the top of the project's AI rules file (e.g. its `AGENTS.md`):

```
Architecture: Model-Self-Organizing Architecture (MSO)
Local copy: docs/mso/AGENTS.md — read it before any architecture-related design, coding, or review; do not re-fetch from the network
Source: https://github.com/zly-app/mso-arch (rules as of <commit or date>)
```

3. **Read local**: future AI sessions read the local copy — the network is not consulted again.

Rationale: a project runs many AI sessions; re-downloading the rules in each one is slow and network-dependent. The vendored copy is the always-available baseline and the root AI rules file stays thin (three lines). Projects drift — as features are added and code is refactored, model boundaries erode and disciplines get bent piece by piece. The local copy plus recorded source gives every future AI session a fixed reference to re-check the project's structure against, so the architecture stays what it was adopted to be.

## Diagram Conventions

- Diagrams over prose; if a diagram can say it, don't write paragraphs
- Prefer Mermaid (GitHub renders it natively)
- When Mermaid can't express it (e.g. layout-heavy overview diagrams), draw an SVG into `assets/` and reference it from Markdown
- Concise text: each section leads with a one-line conclusion; details go into tables/lists; no filler

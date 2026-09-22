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
| **Foundation** | Business-agnostic infrastructure (storage/config/logging/utils). Depended on unidirectionally by upper layers |
| **Business-role model** | Answers "who may do what inside a business domain" (e.g. admin/member). An ordinary model inside some aggregate |
| **Shared Model** | Reused by ≥2 aggregates yet **carrying business rules** (unlike the foundation, which has none). Bound by the same three core disciplines |

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
   - Pick one or combine: **budget** (stop when the call budget is exhausted) / **state-machine phase** (each phase only advances forward) / **re-entrancy guard** (in-progress marker; re-entry returns immediately)
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

### 4. Anti-patterns (report on sight during review)

- **God object**: fields and methods of multiple domains share one struct/class; multiple locks share one host
- **Global god-model**: all domains share one model package; a single change ripples across the repo
- **Rules hidden in the business layer**: orchestration code containing "check role/permission/state" decision logic
- **Models poking each other's data**: directly reading or writing other models' internal fields/tables
- **Cycle without termination**: A→B→A with no stopping mechanism
- **A thickening shell**: business logic growing inside the application shell
- **Splitting for its own sake**: a "model" that cannot pass the one-sentence test

## Review Checklist (self-check when writing or reviewing code)

- [ ] Does the new logic have an owner? Which model? Passes the one-sentence test? If not → design the ownership first
- [ ] Do decision rules live in models? Is the business layer only orchestrating?
- [ ] Do cross-model calls go through the other side's public API? Any direct internal-state access?
- [ ] Any new cycle? What is its termination mechanism? Is there a test proving it stops?
- [ ] Did rules leak into the interface layer? Did the shell get thicker?
- [ ] Does the foundation import any upper layer? (it must not)
- [ ] Does each model's file-header contract state: what data it owns / what it provides / **whom it may call** (lateral-call list)?
- [ ] Do shared models (reused across aggregates) still satisfy data ownership + narrow API + termination guarantee?

## Incremental Adoption (legacy systems · behavior-freezing refactor; full test suite green before each next step)

| Step | Action | Boundary |
|---|---|---|
| 0 Inventory | **Four lenses** to exhaust ownerless things: data ownership / rule invariants / life cycles / use-case flows; produce a 100% file→model mapping | Unassignable files go to a "pending" column with a reason |
| 1 Document | Write a contract per model (owns / provides / allowed-call list) | No structural code changes |
| 2 In-package file groups | Group files by model; write file-header contracts | No new packages, no cross-package file moves |
| 3 Promote to real packages | Promote file groups to independent packages | Models on a cycle merge into one package or cut the cycle with an interface; independent commits per aggregate |

Process discipline:
- Guard against over-splitting: **measure first, then split** — line counts and responsibilities decide; record reasons for anything left unsplit
- Test files travel with their source in big file reorganizations; run the full gate (vet/test/fmt/frontend tests) at every step
- The model inventory is a living document: new models go through the four-lens check to prevent omissions

## Adoption & Provenance

When a project adopts MSO, record the provenance at the top of that project's AI rules file (e.g. its `AGENTS.md`):

```
Architecture: Model-Self-Organizing Architecture (MSO)
Source: https://github.com/zly-app/mso-arch (rules as of <commit or date>)
```

Rationale: projects drift — as features are added and code is refactored, model boundaries erode and disciplines get bent piece by piece. The recorded source gives every future AI session a fixed baseline to re-check the project's structure against, so the architecture stays what it was adopted to be.

## Diagram Conventions

- Diagrams over prose; if a diagram can say it, don't write paragraphs
- Prefer Mermaid (GitHub renders it natively)
- When Mermaid can't express it (e.g. layout-heavy overview diagrams), draw an SVG into `assets/` and reference it from Markdown
- Concise text: each section leads with a one-line conclusion; details go into tables/lists; no filler

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
# Build
go build ./cmd/ainovel-cli

# Run all tests
go test -count=1 ./...

# Run tests for a single package
go test -count=1 ./internal/flow/...

# Run a single test function
go test -count=1 -run TestRoute ./internal/flow/

# Vet
go vet ./...

# Format check (CI gates on this)
test -z "$(gofmt -l .)"

# Race tests (critical state paths only)
go test -race -count=1 ./internal/host ./internal/store ./internal/tools

# Regenerate model registry (after editing gen_models.go)
go generate ./internal/models/...
```

CI runs on Ubuntu and Windows. Always set `GOWORK=off` when running Go commands (the project does not use workspaces).

## Architecture: Deterministic Engine + Semantic Arbiter + Autonomous Workers

This is a fully automatic AI novel creation engine. The core design principle is **"fact layer is deterministic, semantic layer is autonomous"**:

```
Entry (TUI / headless)
  ↓
Host — lifecycle, intervention, event projection, model management
  ├─ Engine — deterministic loop: LoadState → Route → run Worker → repeat
  ├─ Arbiter — LLM-as-function for semantic decisions (plan start, intervention triage, deadlock)
  └─ Workers (architect / writer / editor) — autonomous LLM loops
       ↓
Tools — atomic single-file IO + checkpoints (only interface to Store)
       ↓
Store — file-system facts (Progress, Checkpoints, Drafts, Outlines, etc.)
```

### Two-Planar Symmetry (critical invariant)

Every decision point follows exactly one of two shapes — never invent a third:

| Plane | Mechanism | Validation |
|-------|-----------|------------|
| **Deterministic** | `flow.LoadState` → `flow.Route` → `*Instruction` | Exhaustive combinatorial spec tests |
| **Semantic** | `arbiter.Collect*` → `arbiter.Decide*` → `XxxDecision` | `decisions.jsonl` audit trail + eval regression |

`Route` is a **pure function** (no IO, no Store access) — it reads `State` and returns the next `Instruction` or `nil` (meaning: defer to Engine for plan-start fallback or natural stop). Route covers 13 priority-ordered branches: complete → foundation gap-fill → rewrite queue → reviewing/steering dormancy → arc-end post-processing (review → summary → volume summary → expand arc → append volume) → global review → outline exhaustion → normal chapter writing.

### Key Packages

| Package | Role |
|---------|------|
| `internal/domain` | Shared types (Progress, Outline, Chapter, Checkpoint). Zero internal deps. |
| `internal/flow` | Pure-function Router + State loader. Read by Engine, tested exhaustively. |
| `internal/host` | Host lifecycle, Engine loop, budget/advance-gate/book-lock policies, observer events, import/simulation pipelines. |
| `internal/arbiter` | Semantic decision functions (plan start, intervention triage, failure/deadlock). Each scenario = one Collect+Decide pair + typed Decision struct with Validate. |
| `internal/agents` | Worker assembly (build.go). Subagent construction with role models, prompt cache keys, context managers, stop guards. |
| `internal/tools` | All file IO tools (novel_context, plan/draft/commit_chapter, save_review, save_foundation, etc.). Each runs atomically; commit uses a persistent Saga (PendingCommit). |
| `internal/store` | File-system persistence. Each sub-store manages one artifact type (Progress, Drafts, Checkpoints, Summaries, etc.). |
| `internal/llmcontract` | Structured output enforcement — static Contract → capability detection → native JSON schema or prompt-contract fallback → decode + validate + self-heal. |
| `internal/bootstrap` | Config loading (two-layer merge: `~/.ainovel/` + `./.ainovel/`), model set construction, first-run setup wizard. |
| `internal/diag` | Read-only diagnostics across 4 dimensions (flow/quality/planning/context). Produces findings and a sanitized export for bug reports. Never mutates state. |
| `internal/rules` | User-defined writing rules (forbidden chars/phrases, fatigue words). Loaded from `~/.ainovel/rules/` and `./.ainovel/rules/`, checked mechanically at commit time, consumed by Editor. |
| `internal/eval` | Offline evaluation harness (`ainovel-cli eval` subcommand). Loads test cases, runs A-B comparisons, grades output, generates reports. |
| `internal/errs` | Sentinel errors (`ErrPhaseTransition`, `ErrFlowTransition`, `ErrToolPrecondition`, etc.). |
| `internal/entry/tui` | Bubbletea TUI. |
| `internal/entry/headless` | Headless (non-interactive) mode for servers/CI. |

**Dependency direction**: `entry → host → agents/arbiter → tools → store → domain`. `flow` sits above store but below host (pure policy). `diag` subscribes to host events + reads store read-only. `errs` is importable by any layer.

### State Model

**Phase** (monotonic, forward-only): `init → premise → outline → writing → complete`

**Flow** (cycles within writing phase): `writing ⇄ reviewing ⇄ rewriting ⇄ polishing ⇄ steering`

All transitions are validated by `domain.CanTransitionPhase` / `domain.CanTransitionFlow`. Invalid transitions return `errs.ErrPhaseTransition` / `errs.ErrFlowTransition`.

**Progress** (`meta/progress.json`) is the single source of truth for "where are we". **Checkpoints** (`meta/checkpoints.jsonl`) record every successful tool execution with `Scope+Step+Digest` — duplicates are idempotent (same digest = no new row). **RunMeta** (`meta/run.json`) holds user intent (PlanningTier, PendingSteer, AdvanceMode).

### Writer Chapter Pipeline (fixed order)

1. `novel_context` — load context (summaries, foreshadowing, character states, style rules, related chapter recommendations)
2. `read_chapter` — re-read prior chapters for voice/rhythm
3. `plan_chapter` — design this chapter's goal, conflict, emotional arc
4. `draft_chapter` — write the full chapter body
5. `check_consistency` — cross-check against state data (must be after draft)
6. `commit_chapter` — finalize, write fact fields (`arc_end`, `next_chapter`, feedback pool, rule violations)

### Long-Form Novel Support (Layered Mode)

When `Progress.Layered = true`, the outline uses Volume → Arc → Chapter hierarchy with rolling planning:
- Initial: 2 volumes planned, arc 1 fully expanded with detailed chapters
- At arc end: Editor reviews → generates arc summary → Architect expands next arc
- At volume end: Editor reviews → generates volume summary → Architect decides: append_volume / final volume / complete_book
- `StoryCompass` provides the ending direction, updated at volume boundaries

### Crash Recovery

Recovery is stateless: on restart, Engine reads Store, Route re-derives the next instruction, and re-running a completed step is safe (checkpoint digest idempotency). `PendingCommit` Saga handles mid-commit recovery. `PendingSteer` (single in-flight slot in RunMeta) protects intervention from crashes.

### Book Locking

Each book directory uses a file lock (`.ainovel.lock` via `flock`) to prevent concurrent process conflicts. The `bookLease` is held for the Host's entire lifetime.

### Four Iron Laws

1. **Tools return only facts, never cross-scheduling instructions** — `commit_chapter` returns structured fields (`arc_end`, etc.), not `[SYSTEM]` directive strings
2. **Flow Router handles routing, Engine handles execution** — Route is a pure function; Engine runs Workers programmatically (not via LLM tool forwarding)
3. **Semantic decisions go through Arbiter, every decision is persisted** — `decisions.jsonl` audit trail, replayable offline
4. **Hard-code boundaries, never hard-code unenumerable semantic judgments** — code enforces provable invariants (permissions, ordering, idempotency); creative judgment belongs to Workers/Arbiter

### Config System

Two-layer merge: `~/.ainovel/config.json` (global) ← `./.ainovel/config.json` (project-level, overrides). Config is written back to the active layer (`/config`, `/model` commands). Supports multiple providers, per-role model assignments (`architect`/`writer`/`editor`), budget policies, and notification hooks for unattended runs.

### Structured Output (llmcontract)

The `llmcontract.Contract` defines a static schema. `Execute` handles capability detection (native JSON schema vs prompt-contract fallback), request retry with exponential backoff, schema/DTO decoding, and feedback-based self-healing. The Arbiter exclusively uses this path for structured decisions.

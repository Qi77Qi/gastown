# Gas Town Retrospective: Lessons Learned, Pitfalls, and Efficiency Tips

> **Date:** 2026-03-27
> **Scope:** Everything learned building and operating Gas Town through March 2026
> **Audience:** Future Gas Town operators and contributors

---

## Table of Contents

1. [Worktree Management](#worktree-management)
2. [Branch Management](#branch-management)
3. [Refinery and Merge Flow](#refinery-and-merge-flow)
4. [Bare Repo Sync](#bare-repo-sync)
5. [Pushing Changes](#pushing-changes)
6. [Shallow Clone and Refspec Limitations](#shallow-clone-and-refspec-limitations)
7. [Pitfalls Encountered](#pitfalls-encountered)
8. [Efficiency Tips](#efficiency-tips)
9. [Architecture Insights](#architecture-insights)

---

## Worktree Management

### How Worktrees Are Organized

Gas Town uses three distinct worktree types, each with different isolation guarantees:

| Type | Location | Git Model | Purpose |
|------|----------|-----------|---------|
| **Mayor** | `<rig>/mayor/rig/` | Canonical clone | Shared base for all branching |
| **Polecat** | `<rig>/polecats/<name>/` | Git worktree (from `.repo.git`) | Ephemeral worker sessions |
| **Crew** | `<rig>/crew/<name>/` | Full clone | Human developer workspaces |
| **Refinery** | `<rig>/refinery/rig/` | Worktree or clone (configurable) | Merge queue processing |

### Key Lesson: Worktrees Share Object Storage

All polecat worktrees branch from `.repo.git` (the bare repo). This enables fast
spawning — creating a new polecat is `git worktree add`, not `git clone`. But it
means **mutations to the bare repo affect all worktrees**. If you corrupt refs in
`.repo.git`, every polecat breaks simultaneously.

### Pitfall: Relocated Worktree Paths After Machine Migration

**Problem:** After rsync/move between machines (e.g., macOS `/Users/bob` → Linux
`/home/bob`), worktree `.git` files contain absolute paths to old locations,
breaking all git operations.

**Root causes:**
1. Git worktree `.git` files store absolute paths to the bare repo
2. Scans only recognized rig directories, missing cross-rig worktrees (deacon/dogs)

**Fix (commit `b01e89d2`):**
- `gt doctor` now infers correct `.repo.git` path by extracting rig name from stale path
- Scans `deacon/dogs/<name>/<rigname>/` for cross-rig worktrees
- Manual worktree registration fallback when `git worktree add` fails
- Reports relocated paths clearly: "relocated (`/Users/bob/gt` → `/home/bob/gt`)"

**Prevention:** Always run `gt doctor` after moving a Gas Town installation between
machines. It detects and repairs stale absolute paths.

### Pitfall: `.venv` Permission Issues Blocking Cleanup

**Problem:** Python `.venv` directories in worktrees sometimes have restrictive
permissions that prevent `gt` from cleaning up polecat sandboxes.

**Workaround:** Use `chmod -R u+w` on the worktree before cleanup, or configure
polecats to avoid creating `.venv` in their worktree root.

---

## Branch Management

### Polecat Branch Naming

Each polecat session gets a unique branch: `polecat/<name>/<issue>@<hash>`.

- Branches are created from `origin/<default-branch>` (usually `main`)
- The `polecat_branch_template` rig config controls the pattern
- Branches are ephemeral — they exist only for the lifetime of the work

### Pitfall: Polecats Defaulting to `main` Instead of `target_branch`

**Problem:** When a rig has a non-main target branch (e.g., a release branch),
polecats would still branch from `main` unless explicitly told otherwise.

**Fix (commit `c0f42e0f`):** The `base_branch` formula variable is now propagated
through to MR target in `gt done` and `gt mq submit`. When dispatching work,
always set `--var base_branch=<target>` if the rig's target isn't `main`.

**Rule of thumb:** Formula variables (`--var base_branch=X`) control per-molecule
behavior. Rig config (`target_branch`) controls rig-wide defaults. They are
**different systems** — a formula var overrides the rig config for that specific
molecule.

### Pitfall: Polecat Branches Lost Before Refinery Merge

**Problem:** `gt done` was deleting the polecat branch before the refinery had a
chance to merge it. The work vanished.

**Fix:** `gt done` now creates the MR bead and pushes the branch to origin
**before** any cleanup. The refinery picks up the branch from the remote, not
from the local bare repo. Branch deletion only happens after merge confirmation.

**Rule:** Never delete a branch that hasn't been merged or pushed to origin.

---

## Refinery and Merge Flow

### How the Merge Queue Works

```
Polecat completes work
    → gt done (pushes branch, creates MR bead, nudges refinery)
    → Refinery picks up MR from queue
    → Runs gate checks (build, test, lint, typecheck)
    → If pass: squash-merge to target branch, push, mark MERGED
    → If fail: send FIX_NEEDED back to polecat
```

### Gate Configuration

Gates run in two phases:
- **Pre-merge gates:** Run on the branch before merging (build, lint, typecheck)
- **Post-squash gates (commit `9ae750dc`):** Run after squash but before push
  — catches issues that only appear in the squashed commit

```go
type GateConfig struct {
    Cmd     string        `json:"cmd"`
    Timeout time.Duration
    Phase   GatePhase     // "pre-merge" or "post-squash"
}
```

### Key Config Options

| Config Key | Purpose | Default |
|------------|---------|---------|
| `merge_strategy` | How to merge (squash, merge, rebase) | squash |
| `push_after_merge` | Auto-push after successful merge | true |
| `on_conflict` | What to do on merge conflict | "assign_back" |
| `stale_claim_timeout` | How long before unclaimed MRs timeout | 30 minutes |

### Lesson: The Witness Bottleneck

**Problem (documented in `polecat-self-managed-completion.md`):** The witness
became a serial bottleneck in completion processing:
- Witness patrol cycle takes 30-90 seconds
- Only one completion processed per cycle
- N completions queue up → N × patrol-cycle latency

**Root cause:** Two well-intentioned changes (persistent polecat model + nudge-over-mail)
accidentally centralized all completion discovery in the witness patrol loop.

**Solution direction:** Polecats set `agent_state=idle` directly and nudge the
refinery with `MERGE_READY` themselves. The witness becomes an observer that
handles anomalies (zombies, stuck agents, dirty state) rather than a mandatory
checkpoint in every completion.

**Takeaway:** Be cautious about adding mandatory serial checkpoints to hot paths.
Even small per-item latency compounds badly at scale.

---

## Bare Repo Sync

### The Core Problem (commit `b4ec5c7d`)

After the refinery merges and pushes to origin, the bare repo (`.repo.git`)
remote tracking refs become stale. New polecats branch from outdated code because
they use `origin/<branch>` from the bare repo.

### The Fix

Added `syncBareRepoRefs()` that fetches origin in the bare repo after each
successful merge. Also wired up `syncCrewWorkspaces()` to keep crew clones in sync.

### Rule

**After any mutation to origin (push, merge), sync the bare repo refs.**
This is not optional. Without it, new polecats start from stale code, producing
conflicts or duplicate work.

---

## Pushing Changes

### The Golden Rule: Always Push from the Right Place

| Want to... | Push from | NOT from |
|------------|-----------|----------|
| Submit polecat work | `gt done` (handles push) | Direct `git push` from mayor |
| Update crew workspace | Crew worktree | Mayor worktree |
| Emergency hotfix | Crew worktree → `git push origin main` | Mayor rig root |

### Pitfall: Landing Work in `mayor/rig` Instead of Crew

**Problem:** Making changes in the mayor's rig directory doesn't make them visible
to human developers. The mayor rig is for coordination, not for landing code that
users need to see.

**Rule:** Code that users need goes through the polecat → refinery → main pipeline,
or directly through a crew worktree.

### Pitfall: Merging to `main` Instead of the Checked-Out Branch

**Problem:** Running `git merge` or `git push origin main` from a polecat sends
work directly to main, bypassing the refinery's gate checks entirely.

**Rule:** Polecats NEVER push to main. They push their feature branch and let the
refinery merge after gates pass.

---

## Shallow Clone and Refspec Limitations

### Narrow Fetch Refspec

Crew worktrees use narrow refspecs to avoid fetching the entire repository
history. This means:

- Only branches matching the refspec pattern are available locally
- `git fetch origin <specific-branch>` may fail if it's outside the refspec
- Feature branches created by polecats may not be visible to crew worktrees

### Pitfall: `force-with-lease` Failing Due to Stale Tracking Refs

**Problem:** `--force-with-lease` compares against local tracking refs. If the
bare repo or worktree hasn't fetched recently, the local ref is stale, and the
push is rejected even though no one else modified the branch.

**Fix:** Always `git fetch origin` before `git push --force-with-lease`. The
refinery now does this automatically.

### Pitfall: Crew Worktrees Not Fetching Feature Branches

**Problem:** Narrow refspec means `git fetch` only pulls branches matching the
pattern (usually `refs/heads/main`). Polecat feature branches are invisible.

**Workaround:** Explicitly fetch the branch: `git fetch origin polecat/furiosa/gt-xxx`.
Or widen the refspec temporarily if you need to inspect polecat work.

---

## Pitfalls Encountered

### Cross-Rig Convoy Limitations

**Problem:** Convoys (batch work tracking across rigs) can't resolve beads from
other rigs because each rig has its own Dolt database.

**Workaround:** Use town-level beads (`hq-*` prefix) for cross-rig coordination.
Rig-level beads stay in their rig. The `bd` routing system handles prefix-based
routing automatically.

### `bd` Agent State Warnings

**Problem:** Non-critical but noisy warnings about agent state appear frequently,
cluttering logs without indicating actual problems.

**Context:** These are typically caused by polecats transitioning states faster
than the witness can scan them. The warning is accurate (the state changed) but
not actionable.

**Fix (commit `4f523f1b`):** Skip done/nuked polecats in crash detection. Also
improved (`ffa1e6a8`) to skip crash alerts for polecats that completed normally.

### Tmux Interaction Edge Cases

**Problem (commit `853f9e93`):** Claude Code's "Rewind" menu overlay blocks nudge
delivery because the tmux pane is in a non-standard state.

**Fix:** Detect the rewind menu and dismiss it before delivering the nudge.

**Problem (commit `f981ad17`):** Multiple nudges arriving simultaneously get
interleaved, corrupting the delivery.

**Fix:** Added `flock`-based cross-process lock to serialize nudge delivery.

### Dolt Fragility

Dolt is the data plane for all beads, mail, and agent state. It is a single
server on port 3307. Key lessons:

1. **Never `rm -rf` Dolt data directories.** Use `gt dolt cleanup` instead.
2. **Test pollution is real.** Orphan databases (`testdb_*`, `beads_t*`) accumulate
   and degrade performance. Run `gt dolt cleanup` regularly.
3. **Collect diagnostics before restarting.** A blind restart destroys evidence of
   hangs. Use `kill -QUIT` for goroutine dumps first.
4. **Hyphens in database names** caused validation failures (commit `c7b3f8d9`).
   The regex was fixed to allow them.
5. **Startup timeout scales with database count** (commit `1b27cef4`): 5s per DB.
   With many orphan DBs, startup could timeout before it was ready.

### The Idle Polecat Heresy

**Problem:** Polecats that complete work but don't run `gt done` become zombies.
The witness assumes they're still working. The system stalls.

**Fix:** Multiple layers of enforcement:
- CLAUDE.md instructions (commit `d08547ef`, `afd703b0`)
- `/done` slash command (commit `c8d36a7d`)
- Stop hook safety net that catches session exit without `gt done`
- Witness zombie patrol that detects and restarts idle polecats

**Takeaway:** For autonomous agents, "always run X at the end" needs mechanical
enforcement, not just instructions. Agents will forget, crash, or run out of
context. Build safety nets.

---

## Efficiency Tips

### When to Sling vs. Fix Directly

- **Sling** (dispatch to a polecat) when the fix is well-scoped, can be described
  in a bead, and doesn't require interactive debugging. Polecats excel at
  "implement this spec" work.
- **Fix directly** (from crew) when you need to iterate interactively, the problem
  is vague, or you need to inspect runtime state. Don't force a polecat to do
  exploratory work.

### Rig Config Setup Checklist

When setting up a new rig, configure these in order:

1. **`target_branch`** — What branch polecats target (usually `main`)
2. **`merge_strategy`** — How the refinery merges (`squash` recommended)
3. **`push_after_merge`** — Whether refinery auto-pushes (usually `true`)
4. **`refinery_worktree`** — Whether refinery uses its own worktree (recommended for isolation)
5. **`max_polecats`** — Concurrency limit (default: 10, start lower)
6. **`polecat_branch_template`** — Branch naming pattern (default is fine)

### Formula Variables vs. Rig Config

These are **separate systems** that serve different purposes:

| System | Scope | Set by | Example |
|--------|-------|--------|---------|
| **Formula variables** (`--var`) | Per-molecule | Dispatcher (mayor/witness) | `base_branch=develop` |
| **Rig config** | Rig-wide | `gt rig config set` | `target_branch=main` |

Formula vars override rig config for their specific molecule. If you set
`--var base_branch=develop` on a molecule, that polecat branches from `develop`
even if the rig's `target_branch` is `main`.

### Convoy Management

Convoys track batch work (e.g., "update all rigs to v2"). Tips:

- Use town-level beads for convoy coordination (they're cross-rig)
- Each convoy leg maps to one rig's work
- The mayor creates and monitors convoys; polecats do the legs
- Don't mix convoy beads with implementation beads

### OpenSpec for Planning Before Coding

For significant changes (new capabilities, architecture shifts, breaking changes):

1. Create an OpenSpec proposal first (`/openspec:proposal`)
2. Get it reviewed and approved
3. Then implement via `/openspec:apply`

This prevents wasted work on approaches that won't be accepted.

### Communication Hierarchy

From cheapest to most expensive:

1. **`gt nudge`** — Free, ephemeral, no Dolt overhead. Use for routine pings.
2. **`gt mail send`** — Creates a permanent bead + Dolt commit. Use for messages
   that must survive session death (handoffs, escalations).
3. **`gt escalate`** — Creates a tracked escalation bead with severity routing.
   Use when blocked.

**Rule:** Default to nudge. Only use mail when persistence matters.

### Running `gt doctor` Regularly

`gt doctor` catches many problems before they cascade:
- Stale worktree gitdir paths
- Orphan tmux sessions
- Stale SQL server info files
- Hook sync issues
- Rig config mismatches

Run it after any unusual event (crash, migration, manual git operations).

---

## Architecture Insights

### The Steam Engine Metaphor

Gas Town is designed as a steam engine:
- **Polecats are pistons** — they receive work, execute, and push output
- **The refinery is the crankshaft** — converts polecat output into merged code
- **The witness is the governor** — monitors pressure, handles anomalies
- **The mayor is the engineer** — sets direction and dispatches fuel (work)

The key insight: **throughput depends on pistons firing, not on coordination.**
Every piece of coordination overhead (witness checkpoints, mail messages, Dolt
commits) reduces throughput. Minimize the hot path.

### Two-Level Beads: Why It Matters

Town beads (`hq-*`) and rig beads (project prefix) live in separate Dolt
databases. This is intentional:

- **Isolation:** A rig's Dolt issues don't affect town coordination
- **Routing:** `bd` auto-routes based on prefix, so commands Just Work
- **Scope:** Town beads are for strategy; rig beads are for implementation

### Agent State Machine

Every agent follows a lifecycle tracked in its agent bead:

```
spawning → running → idle → (reuse or nuke)
                  ↘ done → idle (self-managed completion)
                  ↘ stuck → (witness intervention)
                  ↘ crashed → (witness restart)
```

**Key invariant:** An agent's `agent_state` in its bead is the source of truth.
If the state says `running` but the tmux session is dead, the witness detects
and recovers.

### The Capability Ledger

Every completion is recorded. Every bead closed, every MR merged, every escalation
handled — it all goes into the ledger. This serves two purposes:

1. **Accountability:** You can trace exactly what each agent did
2. **Evidence:** Proves that autonomous agent execution works at scale

The ledger shows trajectory, not snapshots. A single bad completion doesn't
define an agent. What matters is the pattern over time.

---

## Summary of Top Lessons

1. **Always sync bare repo refs after mutations** — stale refs cascade into every worktree
2. **Never bypass the refinery** — pushing directly to main skips all safety gates
3. **Build mechanical safety nets** — instructions alone can't prevent agent failures
4. **Minimize serial checkpoints on hot paths** — coordination overhead kills throughput
5. **Run `gt doctor` after anomalies** — it catches problems before they cascade
6. **Default to nudge over mail** — every mail creates Dolt overhead
7. **Formula vars and rig config are separate systems** — know which one to use
8. **Dolt is fragile** — collect diagnostics before restarting, clean up test pollution
9. **Worktree paths are absolute** — migrations between machines break them
10. **`gt done` is not optional** — enforce it mechanically, not just with instructions

# Planning Principles

> **Purpose**: Principles for designing robust system specs. Use these as a
> validation checklist when writing or reviewing any spec.
>
> **Origin**: Extracted from reviewing 22 specs across sessions 1-3 of the
> opex project. These patterns recurred often enough to formalize.
>
> **Coverage lattice**:
> - P1 (Happy Path) + P2 (Entry/Exit) + P8 (No Silent Failures) → catch **missing behavior**
> - P3 (Boundaries) + P4 (Parallel) → catch **interaction bugs**
> - P5 (Name the Mechanism) → catches **vague design**
> - P6 (Limits) → catches **runaway behavior**
> - P7 (Cross-Reference) → catches **consistency drift**
> - P9 (Stability Over Wasted Work) → catches **unsafe shutdown/cleanup**

---

## Principle 1: Happy Path Is Not a Plan

**The happy path is the sketch. Edge cases and failure modes are the actual
design.**

A spec that only describes what happens when everything works is incomplete.
Real system complexity lives in: What happens when step 4 fails? What happens
when the network is down at step 7? What happens when two things that
"shouldn't" happen simultaneously do?

### Validation checklist
- [ ] For every state transition, is there a failure path defined?
- [ ] For every external dependency (API, database, network), is the
      unavailability case handled?
- [ ] For every agent/container, is the crash/OOM case handled?
- [ ] Is there a "what if this takes forever?" timeout for every async
      operation?

### Anti-pattern
Writing a 20-step walkthrough that reads like a demo script. If every step
says "then X happens successfully", the spec is a happy-path narrative, not
a design.

---

## Principle 2: Every State Needs an Entry and an Exit

**If a state exists in a lifecycle diagram, the spec must define how you get
into it AND how you get out of it.**

It's common to define states like `CANCELLED` or `NEEDS_HUMAN` in a lifecycle
table but never describe the triggers that enter or exit those states. A state
without a defined entry is dead code in the design. A state without a defined
exit is a trap.

### Validation checklist
- [ ] For every state in every lifecycle diagram: what event enters it?
- [ ] For every state: what event exits it? (Including: can it exit at all?)
- [ ] For terminal states (`COMPLETED`, `FAILED`, `CANCELLED`): what cleanup
      occurs on entry?
- [ ] For waiting states (`NEEDS_HUMAN`, `BLOCKED`): what is the timeout?
      What happens when the timeout expires?

### Anti-pattern
A lifecycle diagram with 7 states but only 3 of them appear in the
walkthrough. The other 4 are "defined" but unspecified.

---

## Principle 3: Boundaries Must Be Explicit

**When two components interact, the spec must say exactly who does what.
"The system does X" is not a spec — which component, which API, which
direction?**

Ambiguity at component boundaries is the #1 source of integration bugs.
If the spec says "TUI publishes to Redis" but the architecture says TUI
connects to the API server (which writes to Redis), the spec has a boundary
violation that will surface as a real bug.

### Validation checklist
- [ ] Every interaction names the specific source and target component
- [ ] Data flow direction is explicit (who calls whom, who publishes, who
      subscribes)
- [ ] No component reaches past its boundary (e.g., TUI should never touch
      Redis directly)
- [ ] Shared resources (database, queue) have clear ownership and access
      patterns

### Anti-pattern
"The review score is attached to the respawned Leonard." Attached how?
Environment variable? Database record? Redis message field? The verb
"attached" hides a design decision.

---

## Principle 4: Parallel Means Conflicting

**If two things can run in parallel, they can conflict. The spec must define
the conflict resolution strategy.**

Parallelism is never free. If two tasks run in parallel and both modify
overlapping files, they will conflict. If two pipelines run in parallel and
both try to merge into `main`, they will conflict. The spec must address:
what detects the conflict, what resolves it, and what happens when
auto-resolution fails.

### Validation checklist
- [ ] For every parallel operation: what shared state can they collide on?
- [ ] Is there a conflict detection mechanism?
- [ ] Is there an auto-resolution strategy?
- [ ] Is there a human escalation path when auto-resolution fails?
- [ ] Are parallelism limits defined? Per-pipeline or global?

### Anti-pattern
A diagram showing "Feature A" and "Feature B" running in parallel with no
discussion of what happens when they both try to merge to `main` at the
same time.

---

## Principle 5: Name the Mechanism, Not the Outcome

**Specs should describe HOW something happens, not just THAT it happens.
"Auto-resolution" is an outcome. "Git's built-in merge + LLM fallback for
semantic conflicts" is a mechanism.**

Vague verbs hide missing design decisions. Words like "handles", "manages",
"auto-resolves", "processes", and "coordinates" often indicate that the spec
author hasn't decided how the thing actually works. Every verb should be
traceable to a concrete implementation path.

### Validation checklist
- [ ] Are there any vague verbs? ("handles", "manages", "processes",
      "coordinates", "auto-resolves")
- [ ] For every action: could a developer implement this from the spec alone,
      without guessing?
- [ ] Are the tools/libraries/APIs named? ("uses GitHub API `POST /merges`"
      vs "merges the branch")
- [ ] Are data formats specified? ("JSON with fields X, Y, Z" vs "sends the
      data")

### Anti-pattern
"Richelieu attempts auto-resolution." What does this mean? Git's built-in
three-way merge? An LLM reading the conflict markers? A heuristic? The spec
must say.

---

## Principle 6: Limits Prevent Runaway

**Every loop, retry, and queue must have a bound. Unbounded operations are
bugs waiting to happen.**

If Leonard implements, Katherine requests changes, Leonard re-implements,
Katherine requests changes again... how many times? Without a max cycle
count, you get infinite loops. Without queue depth limits, you get memory
exhaustion. Without timeouts, you get zombie processes.

### Validation checklist
- [ ] Every retry has a max count
- [ ] Every loop has a termination condition
- [ ] Every queue has a depth limit or backpressure strategy
- [ ] Every async wait has a timeout
- [ ] Every timeout has a defined consequence (what happens when it fires?)

### Anti-pattern
"Katherine requests changes. Leonard pushes fixes. Katherine re-reviews."
No mention of what happens after the 5th cycle, or the 50th.

---

## Principle 7: Cross-Reference or Contradict

**When the same concept appears in multiple specs, the specs must either
cross-reference each other or they will inevitably contradict.**

A project with 22 spec files will have the same concept (e.g., worktree
paths, branch naming, parallelism limits) mentioned in 3-5 different specs.
If each spec defines the concept independently, they will drift apart over
time. The fix: one spec owns the definition, all others reference it.

### Validation checklist
- [ ] For every concept: which spec is the canonical owner?
- [ ] Do other specs reference the canonical spec, or redefine the concept?
- [ ] After updating a concept: are all references still consistent?
- [ ] Is there an index of which spec owns which concept?

### Anti-pattern
Spec 01 says the worktree path is `/workspace/.worktrees/`. Spec 05 says
it's `/repo/worktrees/`. Spec 18 says it's configurable. All three are
"correct" in isolation.

---

## Principle 8: No Silent Failures

**The system never gives up autonomously. Every failure must surface to a
human who can extract value from it.**

An automatic `FAILED` terminal state generates zero value. The work is lost,
the context is lost, and the human doesn't even know what happened until
they notice. Instead, every failure should escalate to a human decision
point with full context: attempt history, logs, diffs, failure reasons.
The human then decides: retry with more context, take over manually, or
explicitly abandon. The system's job is to succeed or to bring the human
to an informed decision — never to silently give up.

### Validation checklist
- [ ] Is there any state transition that leads to a terminal failure
      without human involvement?
- [ ] When the system can't proceed, does it surface the problem with
      enough context for the human to act?
- [ ] Is attempt history preserved so the human can understand the
      full sequence of what was tried?
- [ ] Can the human always choose between: retry, take over, or abandon?
- [ ] Does "abandon" require an explicit human action (not a timeout)?

### Anti-pattern
A task fails 3 times and the system marks it `FAILED`. The pipeline
continues with remaining tasks, the human notices a week later that a
task was dropped. No context, no decision, no value.

### Design consequence
This principle eliminates automatic terminal failure states. The only
terminal states are success states (`COMPLETED`, `COMPLETED_EXTERNALLY`)
and explicit human decisions (`CANCELLED`). The system can recognize that
it's stuck (auto-retries exhausted), but "stuck" is not "failed" — it's
"waiting for human guidance."

---

## How to Use These Principles

### When writing a new spec
Run through each principle's checklist before marking the spec as "done."

### When reviewing an existing spec
Use the checklists to generate issues (like the 15 issues we found in
spec 01).

### When resolving spec issues
For each issue, identify which principle(s) it violates. The principle
often points directly to the solution.

### Issue → Principle mapping (spec 01 example)

| Issue | Principle violated |
|-------|--------------------|
| 1. Failure scenarios absent | P1 (Happy Path), P2 (Entry/Exit), P8 (No Silent Failures) |
| 2. Cancellation undefined | P2 (Entry/Exit), P9 (Stability Over Wasted Work) |
| 3. NEEDS_HUMAN incomplete | P2 (Entry/Exit), P8 (No Silent Failures) |
| 4. Post-merge rebase trigger missing | P4 (Parallel = Conflicting) |
| 5. Parallel pipelines undefined | P4 (Parallel = Conflicting) |
| 6. TUI → Redis shortcut | P3 (Boundaries) |
| 7. Nelson validation scope | P5 (Name the Mechanism) |
| 8. Feedback attachment vague | P5 (Name the Mechanism) |
| 9. Merge strategy ambiguous | P5 (Name the Mechanism) |
| 10. Review score semantics | P5 (Name the Mechanism) |
| 11. Max review cycles | P6 (Limits) |
| 12. Auto-resolution vague | P5 (Name the Mechanism) |
| 13. Worktree path inconsistency | P7 (Cross-Reference) |
| 14. Branch naming origin | P5 (Name the Mechanism) |
| 15. Parallelism limits scope | P3 (Boundaries), P6 (Limits) |

---

## Principle 9: Stability Over Wasted Work

**When choosing between stopping quickly and stopping cleanly, always choose
clean. Wasting work is cheap; corrupted state is expensive.**

A cancelled pipeline should not leave behind half-pushed branches, partially
written files, or inconsistent database state. It's better to let a Leonard
finish writing a file that nobody will review, or let a Richelieu complete a
push that nobody will merge from, than to interrupt mid-operation and risk
corruption. The wasted compute is bounded (each agent finishes one atomic
step), while the cost of recovering from corrupted state is unbounded.

This principle applies beyond cancellation — anywhere the system faces a
choice between "fast and risky" vs "slow and safe," default to safe.

### Validation checklist
- [ ] Can any operation be interrupted in a way that leaves shared state
      inconsistent?
- [ ] When the system stops (cancel, shutdown, crash recovery), does it
      wait for in-flight operations to complete?
- [ ] Are write operations (git, database, file system) atomic or guarded
      by cleanup on failure?
- [ ] Is the overhead of "wasted work" bounded and predictable?

### Anti-pattern
Force-killing a container that's mid-`git push` to save 30 seconds of
compute. The branch is now in an unknown state on the remote, and the
recovery cost dwarfs the savings.

### Design consequence
Pipeline cancellation uses cooperative shutdown: the orchestrator stops
dispatching new work, running agents finish their current atomic operation
and exit naturally. No SIGTERM, no timeout, no SIGKILL. The orchestrator
waits as long as needed for all containers to exit cleanly.

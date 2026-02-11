# Red Team: Channel Inbox/Outbox Architecture

> **Created:** 2026-02-11 | **Lead:** Agent | **Mode:** Skeptic / Forensic
> **Subject:** `20260211-channel-inbox-outbox-proposal.md` | **Related:** `spec.md`, `impl/08_concurrency.md`, `impl/09_runtime.md`, `20260211-channel-inbox-outbox-spec-plan.md`, `20260211-channel-inbox-outbox-impl-spec-plan.md`

---

## Bottom Line

**Verdict:** Fix Issues

**Top Vulnerabilities:**
1. **Spec amendment required** — Adding `wait` (and `timeout`) parameters to `/push` with blocking semantics requires explicit spec updates; the proposal must own this as a language-surface change.
2. **Pull scan atomicity** — Concurrent pullers scanning outboxes can race on the same entry. The proposal must require concurrency-safe implementations.
3. **Deadlock risk with waited push** — Two contexts doing waited `/push` to each other = instant deadlock. Mitigated by `timeout` parameter but must be documented.

---

## Stakeholder Roles

| Role | Cares About | Pain Points | Critical Assumptions |
|------|-------------|-------------|---------------------|
| **Script Authors** | Simple, predictable channel semantics | Blocking push changes behavior of existing scripts silently | Push is fire-and-forget (current spec/impl) |
| **Runtime Implementors** | Clear spec, implementable pseudocode, testable contracts | Scanning all outboxes is non-trivial correctness work; new `WAITING_CHANNEL_PUSH` state adds complexity | The impl spec is the truth; new states/flows must be complete |
| **Language Designers** | Spec consistency, extensibility, minimal footprint | New `wait` parameter and blocking push semantics leak into the core language surface | Internal changes don't require spec amendments |
| **Debugger/Tooling Authors** | Inspectability, observable state | More distributed state (per-context outboxes/inboxes) is *harder* to inspect than a single queue | Centralized state is easier to observe |

---

## Attack Surface (Per Role)

**Script Authors:**
- Claims to this role: "This is an internal architecture change, not a language semantics change" (proposal line 12)
- Assumptions by/about role: Existing scripts will work. Push is non-blocking fire-and-forget.
- Dependencies: `/push` current behavior: push-and-continue

**Runtime Implementors:**
- Claims to this role: The pseudocode is complete and correct enough to implement
- Assumptions by/about role: They can implement the outbox scan atomically and correctly
- Dependencies: Clear state machine, no ambiguous flows

**Language Designers:**
- Claims to this role: Spec doesn't need amendment; the external semantics are unchanged
- Assumptions by/about role: The spec's "global pipe" language is compatible with distributed storage
- Dependencies: `spec.md` lines 341-344 defining channels

---

## Critical Findings

### Finding 1: Spec Amendment Required — `/push` Gains Blocking Semantics

**Severity:** High | **Likelihood:** High

**Affects Roles:** Script Authors, Language Designers, Runtime Implementors

**Attack Vector:** The current spec (`spec.md` line 1474-1475) defines `/push` as returning "A nothing" with no named parameters beyond `channel` and `var`. The proposal adds two new named parameters (`wait`, `timeout`) and blocking behavior. Since the spec is actively being revised alongside this proposal, this isn't a "violation" per se — but the proposal must explicitly own that it's a **language-surface change** requiring spec amendment, not merely an "internal architecture change" (proposal line 12).

Specifically, the proposal must plan for:
1. Adding `wait` (boolean, default `true`) named parameter to `/push` in spec
2. Adding `timeout` (double, optional) named parameter to `/push` — usable alongside `wait: true` to prevent indefinite blocking
3. New `WAITING_CHANNEL_PUSH` context state
4. Updated diagnostics for push timeout (`Info: timeout`) and close-while-waiting (`Error: closed`)

> [!NOTE]
> `wait: true` is confirmed as the default. ZOH is a brand new language with no existing scripts, so backward compatibility is not a concern.

**Role-Specific Impact:**
- **Language Designers:** `spec.md` Channel.Push section needs amendment for `wait` and `timeout` parameters.
- **Runtime Implementors:** Must implement a new blocking state (`WAITING_CHANNEL_PUSH`) not present in the current `ContextState` enum.

**Blast Radius:** Spec, all runtimes, potentially existing scripts (depending on default).

**Remediation:** (a) Resolve `wait` default in the proposal explicitly, (b) add `timeout` named parameter to `/push` for use alongside `wait: true`, (c) include spec amendment as an explicit deliverable of this proposal, not an afterthought.

---

### Finding 2: Pull Scan Lacks Atomicity — Concurrent Pullers Can Race

**Severity:** High | **Likelihood:** Medium

**Affects Roles:** Runtime Implementors

**Attack Vector:** The pull flow (proposal lines 198-206) scans outboxes to find the entry with the lowest sequence number. The pseudocode uses `for ctx in runtime.contexts` — scanning *all* contexts — but this is easily fixed: the hub can maintain a `Map<contextId, Queue>` of participating outboxes (populated on `/open`), reducing the scan from O(N total contexts) to O(K participating contexts). 

The **real hazard** is concurrency between multiple pullers. Even with K outboxes in a dictionary:
1. Two concurrent pullers can both find the same lowest-seq entry and both attempt to dequeue it — violating single-dispatch
2. A push occurring mid-scan could insert a lower sequence number that one puller sees and another misses
3. Context termination during scan could remove an outbox mid-iteration

The pseudocode treats this as a simple for-loop but real concurrent implementation needs a lock on the hub-level scan-and-dequeue or an atomic claim mechanism.

**Role-Specific Impact:**
- **Runtime Implementors:** Race conditions between concurrent pullers could cause duplicate delivery or lost messages. The pseudocode undersells this.

**Blast Radius:** Core channel correctness — FIFO single-dispatch guarantee.

**Remediation:** (a) Hub should maintain a participation dictionary of outboxes (not scan all contexts), (b) the scan-and-dequeue must be specified as an atomic operation (hub-level lock or compare-and-swap on the winning outbox entry), AND (c) the proposal must explicitly state that **all hub and outbox data structures must be concurrency-safe** (e.g., concurrent collections, lock-guarded access) — this is a requirement on implementors, not an implementation detail to leave implicit.

---

### Finding 3: Deadlock Risk with Waited Push

**Severity:** Medium | **Likelihood:** Medium

**Affects Roles:** Script Authors, Runtime Implementors

**Attack Vector:** With `wait: true`:
```zoh
:: Context A                     :: Context B
/push <chan_b>, *val_a;          /push <chan_a>, *val_b;
/pull <chan_a>;                  /pull <chan_b>;
```
Both contexts block on push before ever reaching pull. Classic deadlock.

The proposal acknowledges this (line 321) but proposes only "timeout support" or "document as user responsibility."

**Role-Specific Impact:**
- **Script Authors:** Subtle, hard-to-debug hangs in scripts that look reasonable
- **Runtime Implementors:** No deadlock detection mechanism is specified

**Blast Radius:** Any multi-context coordination pattern with bidirectional communication.

**Remediation:** The `timeout` named parameter (see Finding 1) is the primary mitigation — waited push with `timeout` prevents indefinite blocking. The proposal should: (a) add `timeout` as a named parameter on `/push` usable with `wait: true`, (b) document the deadlock risk pattern explicitly, and (c) recommend `timeout` usage in bidirectional communication patterns.

---

### Finding 4: `/open` Flow Incomplete — Missing Hub Registration and Spec Update

**Severity:** Medium | **Likelihood:** High

**Affects Roles:** Runtime Implementors

**Attack Vector:** The proposal's `/open` flow (lines 230-245) creates the hub and initializes context-local outbox/inbox, but has two gaps:

1. **No hub registration:** The hub has no record of which contexts have opened the channel. The pull flow needs to know which contexts have outboxes — without a participation list, it must scan all runtime contexts (proposal line 201: `for ctx in runtime.contexts`).

2. **`/open` behavior change not fully specified:** Under the new model, `/open` must also initialize context-local outbox/inbox and register with the hub. The proposal's open question (line 335-336) asks whether `/push`/`/pull` should auto-create outbox/inbox — this needs resolution.

**Role-Specific Impact:**
- **Runtime Implementors:** Must choose between scanning all contexts (inefficient) or maintaining a participation list (not specified). The `/open` pseudocode needs updating.

**Blast Radius:** Performance, code clarity, and correctness of participation tracking.

**Remediation:** (a) Hub should maintain a `participatingContexts: Map<contextId, {outbox, inbox}>` populated by `/open`, (b) `/open` pseudocode must be updated to register context with hub, (c) resolve whether `/push`/`/pull` auto-register or require explicit `/open`.

---

### Finding 5: Context Termination / Cleanup Is Underspecified

**Severity:** Medium | **Likelihood:** Medium

**Affects Roles:** Runtime Implementors, Script Authors

**Attack Vector:** When a context terminates (or hits `/exit`), the proposal's open question (line 337) acknowledges outbox cleanup is unresolved. But there are several entangled issues:

1. **Orphaned outbox values:** If Context A pushes 10 values and exits, are those values still pullable? If yes, who owns the memory? If no, pullers lose data.
2. **Blocked waited pushers:** If Context A is `WAITING_CHANNEL_PUSH` and the channel closes, the close flow (lines 260-268) only wakes pullers, not pushers. Waited pushers are stranded.
3. **Inbox cleanup:** When a context exits with unread inbox values, are they silently discarded?
4. **Hub deregistration:** When a context exits, it must be removed from all hubs' participation lists (see Finding 4).

**Role-Specific Impact:**
- **Script Authors:** Unpredictable behavior around context lifecycle boundaries
- **Runtime Implementors:** Missing cleanup logic = memory leaks or dangling references

**Blast Radius:** Any pattern where contexts have different lifetimes.

**Remediation:** The proposal must include explicit cleanup flow: (a) close flow must wake waited pushers (the behavior table on line 278 says this but pseudocode omits it), (b) context termination must deregister from all hubs, (c) orphaned outbox values should be discarded (consistent with "data belongs to the context"), (d) stranded waited pushers whose context exits must be woken with error, (e) inbox values discarded on exit. This should be specified as pseudocode in the proposal, not left as an open question.

---

### Finding 6: Inspectability Is a Mixed Improvement

**Severity:** Low | **Likelihood:** Low

**Affects Roles:** Debugger/Tooling Authors

**Attack Vector:** The proposal claims (line 40) improved inspectability via per-context visibility. This is partially true:

- **Improved:** Per-context outbox/inbox gives clear origin tracking (who pushed what, who received what). This is genuinely better for debugging "who sent this value?" questions.
- **Trade-off:** Aggregate channel state ("what's pending in `<events>` across all contexts?") now requires scanning multiple outboxes. With hub participation lists (Finding 4), this is bounded to K contexts, not N.

With the hub maintaining participation dictionaries, the aggregation cost is manageable. The proposal's inspectability claim holds for per-context debugging but should acknowledge the aggregation cost for channel-level views.

**Role-Specific Impact:**
- **Debugger/Tooling Authors:** Better origin tracking, slightly more work for aggregate views

**Blast Radius:** Developer experience (minor).

**Remediation:** Acknowledge the trade-off explicitly in the proposal. Consider adding a hub-level `pendingCount()` helper that sums across participating outboxes for quick aggregate inspection.

---

## Role-Based Assumption Challenges

### Script Authors: "This is just an internal architecture change"
**Challenge:** Adding `wait` and `timeout` parameters to `/push` changes the language surface. The spec defines `/push` with no such parameters.
**Counter-Evidence:** `spec.md` line 1468-1475 — the push verb spec has no `wait` or `timeout` parameters.
**If Wrong:** The proposal ships without spec update, creating spec/impl divergence.
**Action:** Validate — the proposal must include spec amendment as an explicit deliverable. The framing should be corrected from "internal architecture change" to "architecture change with spec-level additions."

### Language Designers: "Spec says channels are global pipes — this is still true"
**Challenge:** `spec.md` line 342 says `<channel>` "servers as a pointer to the underlying data structure uniquely identified by 'channel_name' in the channel-dedicated storage in the runtime." The proposal splits this "underlying data structure" into per-context fragments coordinated by a hub.
**Counter-Evidence:** The "one same underlying data structure" language (line 344) implies a single shared object, not distributed storage + routing.
**If Wrong:** Spec ambiguity leads to divergent implementations.
**Action:** Validate — the spec language should be updated to clarify that the "underlying data structure" includes the routing layer.

### Runtime Implementors: "Sequence numbers ensure global FIFO"
**Challenge:** Sequence numbers ensure ordering only if the scan is atomic. If two pullers scan simultaneously, both could find the same lowest-sequence value. Single-dispatch requires locking.
**Counter-Evidence:** The pseudocode has no locking primitives or concurrency-safety requirements.
**If Wrong:** Values delivered to multiple pullers, violating FIFO single-dispatch.
**Action:** Validate — add explicit concurrency-safety requirement: all hub and outbox operations must use concurrency-safe data structures or explicit locking.

---

## Role-Specific Edge Cases & Failures

### Script Authors: Spec Example Breaks
**Trigger:** The `spec.md` "At a Glance" example (lines 83, 105) uses `/push <stop_idle>, true;` and `/push <ending>, *trust;` — these are fire-and-forget pushes with no guarantee a puller is ready.
**Role Experience:** Under `wait: true` default, the main context blocks on push until `@barista_idle` pulls from `<stop_idle>`. If the barista idle loop already exited (timeout hit), the push blocks forever.
**Recovery:** Difficult — requires rewriting example with `wait: false`.
**Mitigation:** Default `wait: false`.

### Runtime Implementors: Close Flow Missing Pusher Wake-up
**Trigger:** Close a channel while contexts are in `WAITING_CHANNEL_PUSH` state.
**Role Experience:** Waited pushers never wake up — they're stranded in `WAITING_CHANNEL_PUSH` forever.
**Recovery:** Impossible without restart.
**Mitigation:** Close flow pseudocode must iterate `hub.waitingPushers` and wake them with error (the table on line 278 says "wakes all blocked pullers **and pushers**" but the pseudocode omits pushers).

### Runtime Implementors: Inbox Queue Grows Unboundedly
**Trigger:** Fast push path (puller already waiting) delivers directly to inbox. If the puller wakes, processes, then calls `/pull` again — it checks inbox first. But under normal flow, the pull-scan also deposits into inbox. The inbox is never bounded or cleaned.
**Role Experience:** Memory growth for high-throughput channels.
**Recovery:** Possible — add bounds or auto-purge.
**Mitigation:** Document inbox semantics (transient buffer vs. accumulator).

---

## What's Hidden (Per Role)

**Omissions per role:**
- **Script Authors:** Not told that the default push behavior fundamentally changes — buried in open questions
- **Runtime Implementors:** Not told how to handle concurrent pull scans, context termination cleanup, or inbox lifecycle
- **Language Designers:** Not told this requires spec amendment for `wait` parameter

**Tradeoffs per role:**
- **Script Authors:** Gain rendezvous semantics, lose fire-and-forget simplicity
- **Runtime Implementors:** Gain per-context data locality, lose simple single-queue implementation
- **Debugger Authors:** Gain per-context origin tracking, lose single-point-of-truth channel inspection

---

## Scale & Stress (Role Impact)

**At 10x contexts (10→100):**
- **Runtime Implementors:** Pull scan iterates 100 contexts per pull call. With M channels active, each pull is O(100). Contention on outbox access.
- **Script Authors:** More deadlock risk with more contexts doing waited pushes.

**At 100x contexts (100→10,000):**
- **Runtime Implementors:** Pull scan becomes untenable. The "optimize later" suggestion (proposal line 318) becomes "must solve now."

---

## Remediation

### Must Fix (Before Proceeding)
- **Spec amendment plan** (affects: Script Authors, Language Designers) → Add `wait` and `timeout` named parameters to `/push` in spec. Resolve `wait` default. Include spec update as explicit deliverable.
- **Close flow: wake pushers** (affects: Runtime Implementors) → Add `waitingPushers` wake-up loop to close pseudocode → Verify against behavior table claim
- **Concurrency-safety requirement** (affects: Runtime Implementors) → Explicitly require all hub and outbox data structures to be concurrency-safe. Specify scan-and-dequeue as atomic operation.
- **`/open` flow update** (affects: Runtime Implementors) → Update `/open` pseudocode to register context with hub participation dictionary. Resolve whether `/push`/`/pull` auto-register.
- **Context cleanup flow** (affects: Runtime Implementors, Script Authors) → Specify pseudocode for context termination: deregister from hubs, discard outbox/inbox, wake stranded waited pushers with error.

### Should Fix (Before Implementation)
- **Hub-level `pendingCount()` helper** (affects: Debugger Authors) → Provide aggregate inspection across participating outboxes
- **Deadlock documentation** (affects: Script Authors) → Document waited-push deadlock pattern and recommend `timeout` usage

### Monitor
- **Inbox memory growth** (affects: Runtime Implementors) → Revisit if high-throughput patterns emerge

---

## Final Assessment

**Soundness:** Fixable — The core insight (per-context data ownership) is valid, but the execution has gaps in concurrency safety, spec alignment, and completeness.
**Risk:** Medium — No fundamental design flaw, but the blocking-push default and missing pseudocode details could cause real damage if implemented as-is.
**Readiness:** Needs Work

**Per-Role Readiness:**
- **Runtime Implementors:** Not Ready — Pull scan concurrency, close-flow pusher wake-up, `/open` registration, and cleanup are underspecified
- **Language Designers:** Not Ready — Spec amendment needed for `wait` and `timeout` parameters on `/push`
- **Debugger/Tooling Authors:** Ready with Caveats — Better than current if hub participation list exists

**Conditions for Approval:**
- [ ] Include spec amendment as explicit deliverable (`wait`, `timeout` params on `/push`) (for Language Designers)
- [ ] Complete close flow to wake waited pushers (for Runtime Implementors)
- [ ] State concurrency-safety requirement for all hub/outbox structures (for Runtime Implementors)
- [ ] Update `/open` pseudocode with hub registration (for Runtime Implementors)
- [ ] Specify context termination cleanup flow as pseudocode (for Runtime Implementors)
- [ ] Hub tracks participating contexts via dictionary (for Runtime Implementors)

**No-Go If:**
- [ ] No spec amendment planned for `wait`/`timeout` parameters (impacts Language Designers)
- [ ] Close flow does not wake waited pushers (impacts Runtime Implementors — stranded contexts)
- [ ] Context cleanup left as open question (impacts Runtime Implementors — memory leaks)

# Memo: Runtime Design Gaps Against Spec

> **Date:** 2026-03-04
> **Author:** Agent
> **Source Type:** Issue
> **Origin:** Conversation — user asked to examine spec/ vs impl/09_runtime.md for problems

---

## Source

Cross-referencing `spec/` (0_basic, 1_concepts, 2_verbs, 3_runtime) against `impl/09_runtime.md` surfaced nine issues ranging from critical design gaps to minor omissions.

---

## Context

### Critical

**1. No story transition lifecycle.**
The runtime has `Context.terminate()` but no `leaveStory()`. Cross-story `/jump` must: execute story-scoped defers (LIFO), drop story variables, then switch story+checkpoint. Currently these responsibilities would fall entirely on JumpDriver with no runtime support or documentation. Spec refs: `spec/2_verbs.md` L772 (story vars dropped on jump), L1383 (story defers fire on leaving story).

**2. `/call` with `[inline]` races against context cleanup.**
CallDriver suspends with `JoinContext`. On fulfillment, `terminate()` has already called `runtime.removeContext(this)` on the forked context — but `[inline]` needs to read the forked context's variables after termination to copy them back. The `onFulfilled` handler only receives `Completed { lastReturnValue }`, not the variable state. Spec ref: `spec/2_verbs.md` L882.

### Moderate

**3. Push blocking table is misleading.**
Runtime table says `/push` blocks on "wait: true and no puller". Spec says it blocks "until the value is consumed" (rendezvous). A puller can exist without having consumed this specific value — push should still block. Spec ref: `spec/2_verbs.md` L1014.

**4. Channel "unbounded" vs rendezvous tension.**
`spec/1_concepts.md` L112 says "unbounded global pipe". Default push is rendezvous (zero-capacity sync pattern). Reconcilable (`wait: false` is unbounded, `wait: true` adds sync) but the spec's "unbounded" is misleading and the runtime doesn't clarify the dual personality.

**5. Example in `0_basic.md` never calls `/open`.**
The showcase pushes to `<stop_idle>` and `<ending>` without `/open`. Spec says `/push`/`/pull` return `Error: not_found` for non-existent channels. Either channels auto-create on first use or the example is wrong.

**6. `/try` with suspending inner verbs is unspecified.**
`/try /pull <chan>;` — if `/pull` returns `Suspend`, TryDriver must either propagate the suspension (wrapping `onFulfilled`) or forbid suspending verbs inside `/try`. Neither is specified.

**7. Per-context outbox/inbox vs global FIFO pipe.**
Spec describes a "global pipe" (one queue). Runtime introduces per-context outboxes/inboxes plus a ChannelHub. The mapping isn't documented, and FIFO ordering with multiple pushers is ambiguous (global vs per-pusher).

### Minor

**8. `/exit` value not modeled in `terminate()`.**
`/exit value;` accepts a return value (spec L1180) but `terminate()` doesn't set `lastReturnValue`. ExitDriver would need to set it before calling `terminate()` — flow not shown.

**9. Cascading fatals during defer execution.**
If a deferred verb itself produces a fatal, are remaining defers skipped? Not addressed.

---

## Related Projex

- 20260304-context-runtime-coupling-eval.md

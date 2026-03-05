# Async Host Pump (Reference Adapter)

This scenario documents a minimal host-side adapter for async environments that still uses the runtime public API:

- `runtime.tick(deltaTimeMs)` advances time.
- `runtime.resume(handle, value)` fulfills host waits.
- Runtime mutation is serialized per runtime instance.

## Minimal Pump Pattern

```text
AsyncRuntimePump:
    runtime: Runtime
    tickIntervalMs: double
    inbox: AsyncQueue<fn()>

    async runUntilCancelled():
        last = monotonicNowMs()
        while not cancelled:
            action = await inbox.tryDequeueAsync(timeoutMs: tickIntervalMs)

            now = monotonicNowMs()
            dt = now - last
            last = now
            runtime.tick(dt)

            if action != null:
                action()
                # Optional: continue execution immediately without advancing time.
                runtime.tick(0)

    postResume(handle, value):
        inbox.enqueue(() -> runtime.resume(handle, value))
```

## Integration Notes

- Use a monotonic clock for `dt`.
- Keep exactly one mutator loop per runtime/session.
- If you stop periodic ticking, `/sleep` and timeouts become lazy and are only observed on the next `tick()`.
- This pattern is a reference implementation, not a required architecture.

## Related Scenario

For a concrete headless/server-style workload that this adapter can drive, see `impl/scenarios/mud_server.md`.

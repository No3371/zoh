# Red Team: ZOH Spec and Implementation Guides

> **Created:** 2025-02-07 | **Lead:** Agent | **Mode:** Attack/Skeptic/Forensic
> **Subject:** spec.md and impl/ | **Related:** Core language specification and implementation workflow

---

## Bottom Line

**Verdict:** Proceed with Caution

**Top Vulnerabilities:**
1. **Channel race conditions and stale reference attacks** — Channels can be closed and recreated with same name; generation IDs mentioned in impl but not in spec
2. **Unbounded resource consumption** — No limits on context creation, channel depth, collection sizes, or persistent storage
3. **Expression injection via unconstrained interpolation** — `$*var` treats variable content as template, enabling indirect code injection patterns

---

## Stakeholder Roles

| Role | Cares About | Pain Points | Critical Assumptions |
|------|-------------|-------------|---------------------|
| Story Authors | Reliable script execution, predictable behavior | Subtle differences between spec and impl | Variables work as documented |
| Runtime Developers | Clear, unambiguous spec | Underspecified edge cases, conflicting guidance | Impl guides match spec |
| End Users/Players | Stable gameplay, saved progress | Corrupted saves, hangs, crashes | Runtime protects against malformed scripts |
| Security/Operators | No resource exhaustion, no injection | Unbounded loops, memory consumption | Scripts are sandboxed |
| Integrators | Predictable API, cross-runtime compatibility | Implementation-defined behaviors | All runtimes behave identically |

---

## Attack Surface (Per Role)

**Story Authors:**
- Claims: Variables persist correctly, types behave as documented
- Assumptions: Story/context scope separation is reliable, defers execute in order
- Dependencies: Parser handles all syntactic sugar correctly, preprocessor expands macros safely

**Runtime Developers:**
- Claims: Spec is complete and consistent, impl guides faithfully represent spec
- Assumptions: Type coercion is fully specified, error codes are exhaustive
- Dependencies: All edge cases are documented

**Security/Operators:**
- Claims: Scripts cannot hang runtime, consume unbounded resources, or corrupt state
- Assumptions: Channel operations are safe, concurrent access is handled, storage is protected
- Dependencies: Runtime implements resource limits (but none are specified)

**Integrators:**
- Claims: Cross-runtime portability, consistent behavior
- Assumptions: "Implementation-defined" means minor variations only
- Dependencies: All runtimes implement same core verbs with same semantics

---

## Critical Findings

### Finding 1: Channel Race Conditions and Stale References [DONE](closed/20260207-channel-racecond-walkthrough.md)

**Severity:** High | **Likelihood:** Medium

**Affects Roles:** Runtime Developers, Story Authors, Security

**Attack Vector:** 
```zoh
:: Context A
/fork ?, "worker";
/sleep 0.1;
/close <chan>;
/push <chan>, "malicious";  :: Creates NEW channel with same name

:: Context B (worker)
@worker
/pull <chan>; -> *val;  :: Which channel? Old (closed) or new?
```

**Role-Specific Impact:**
- **Runtime Developers:** Must implement generation IDs correctly (mentioned in impl but not spec)
- **Story Authors:** Unexpected behavior when reusing channel names
- **Security:** Possible data leakage or misdirection between contexts

**Blast Radius:** Cross-context communication corruption, state confusion

**Remediation:** 
1. Spec MUST document channel generation IDs explicitly
2. Spec should forbid same-name channel creation after close, OR document exact semantics
3. Consider warning diagnostic when pushing to recreated channel

---

### Finding 2: Unbounded Resource Consumption

**Severity:** High | **Likelihood:** High

**Affects Roles:** Security, Operators, End Users

**Attack Vector:**
```zoh
:: Fork bomb
/loop -1, /fork ?, "bomb";;

:: Memory exhaustion via channel
/loop -1, /push <mem>, "AAAAA...repeated 1MB string";;

:: Infinite list growth
*list <- [];
/loop -1, /append *list, "data";;
```

**Role-Specific Impact:**
- **Security:** Denial of service, runtime crash
- **Operators:** System resources exhausted, affects other tenants
- **End Users:** Game hangs, progress lost

**Blast Radius:** Entire runtime and potentially host system

**Remediation:**
1. Spec should define `maxContexts`, `maxChannelDepth`, `maxCollectionSize` as RuntimeConfig
2. Impl guides mention `maxContexts` but spec does not enforce it
3. Add `timeout` attribute to `/loop` and `/while`
4. Consider script-level resource quotas

---

### Finding 3: Expression Injection via `$*var` Interpolation [DONE](20260207-expression-injection-eval.md)

**Severity:** Medium | **Likelihood:** Medium

**Affects Roles:** Story Authors, Security

**Attack Vector:**
```zoh
:: User input stored in *user_input
*template <- "Hello ${*user_input}!";
/interpolate *template;  :: What if user_input = "World${*secret_var}"?
```

The spec says `$*var` evaluates the variable as a template and interpolates it ONCE. But what about nested references in user-controlled strings?

**Role-Specific Impact:**
- **Story Authors:** Unintended variable exposure
- **Security:** Information disclosure, scope confusion

**Blast Radius:** Variable value leakage, potentially across scopes

**Remediation:**
1. Clarify spec: does interpolation of user-controlled strings recurse?
2. Add `[raw]` attribute to disable interpolation of variable contents
3. Document safe patterns for user input handling

---

### Finding 4: Spec/Impl Inconsistencies

**Severity:** Low | **Likelihood:** High

**Affects Roles:** Runtime Developers, Integrators

**Attack Vector:** N/A — This is a documentation/consistency issue

**Identified Inconsistencies:**

| Topic | Spec Says | Impl Says | Status |
|-------|-----------|-----------|--------|
~~| Channel generation IDs | Not mentioned | `impl/08`: "Generation IDs" for stale reference detection |~~ [DONE](closed/20260207-channel-racecond-walkthrough.md)
~~| `/pull` on closed channel | Error: `closed` | `impl/08`: Returns `PullResult { status: "closed" }` | **VALID**: Impl guide incorrect |~~ [DONE](closed/20260207-fix-spec-impl-inconsistencies-patch.md)
| ~~Map key order in `toString()`~~ | "implementation-defined" | "Order is implementation-defined" (Impl/05) | **INVALID**: Impl/05 is consistent |
| ~~`/foreach` iterator drop~~ | "dropped from **story scope** first" | `context.drop(iteratorName, scope: STORY)` (Impl/07) | **INVALID**: Impl/07 is consistent |
~~| Macro syntax | `|%NAME%|` pipe-delimited | `impl/03`: Intro text mentions `#macro` | **VALID**: Doc text error (needs fix) |~~ [DONE](closed/20260207-fix-spec-impl-inconsistencies-patch.md)

**Role-Specific Impact:**
- **Runtime Developers:** Minor confusion about authoritative source for Macros
- **Integrators:** No impact (Code/Spec are consistent on behavior)

**Remediation:**
- Fix `impl/08` pseudo-code to throw error on closed channel (match Spec)
- Fix `impl/03` introduction text to remove `#macro` reference (match Spec)

---

### Finding 5: Defer Execution Order Edge Cases

**Severity:** Medium | **Likelihood:** Low

**Affects Roles:** Story Authors, Runtime Developers

**Attack Vector:**
```zoh
/defer /close <cleanup>;;
/defer /push <cleanup>, "final";;  :: Defers execute LIFO per spec
:: What happens? close executes first (LIFO), push fails?
```

**Role-Specific Impact:**
- **Story Authors:** Subtle bugs in cleanup ordering
- **Runtime Developers:** Must preserve strict LIFO, but what about errors in defers?

**Blast Radius:** Resource leaks, incomplete cleanup

**Remediation:**
1. Spec should clarify: if a defer fails, do subsequent defers still execute?
2. Add examples showing proper defer ordering patterns
3. Consider `/try` wrapper for defers

---

### Finding 6: Checkpoint Contract Validation Timing

**Severity:** Medium | **Likelihood:** Medium

**Affects Roles:** Story Authors, Runtime Developers

**Attack Vector:**
```zoh
@entry *required_var
:: Author expects *required_var to be validated on entry
:: But when exactly? Parse time? Compile time? Runtime jump?

/fork ?, "entry";  :: *required_var not set in child context
```

**Role-Specific Impact:**
- **Story Authors:** False confidence in contract enforcement
- **Runtime Developers:** Unclear when to validate

**Blast Radius:** Null/nothing reference errors at unexpected times

**Remediation:**
1. Spec says "validation happens" but doesn't specify exactly WHEN
2. Clarify: contracts are validated at jump/fork/call TIME in the TARGET context
3. Add test cases for all navigation verbs + contracts

---

### Finding 7: Type Coercion Ambiguities

**Severity:** Low | **Likelihood:** Medium

**Affects Roles:** Story Authors, Runtime Developers

**Attack Vector:**
```zoh
*val <- 42.9;
/set *int_var, *val;  :: Double to integer: "rounded toward zero" per spec
:: Is this 42 or 43? Spec says "truncate" which is toward zero = 42
:: But what about -42.9? Should be -42, not -43

*str <- "   123   ";
/parse *str, "integer";  :: Is whitespace trimmed? Not specified
```

**Role-Specific Impact:**
- **Story Authors:** Unexpected numeric conversions
- **Runtime Developers:** Edge case implementation differences

**Blast Radius:** Subtle numeric bugs, cross-runtime inconsistencies

**Remediation:**
1. Add explicit examples for negative truncation
2. Specify whitespace handling in `/parse`
3. Consider `/round` verb for explicit rounding control

---

## Role-Based Assumption Challenges

### Story Authors: "Variables just work like other languages"
**Challenge:** ZOH has story/context scope separation that shadows like lexical scope but persists differently
**Counter-Evidence:** Context variables persist across story jumps; story variables are cleared
**If Wrong:** Authors may lose state unexpectedly on `/jump`
**Action:** Validate — Add clear scope indicators in diagnostics, consider `/scope` query verb

### Runtime Developers: "Impl guides are authoritative"
**Challenge:** Spec and impl guides sometimes conflict (especially macros)
**Counter-Evidence:** Two completely different macro syntaxes documented
**If Wrong:** Implementations may not be spec-compliant
**Action:** Reject — Spec MUST be reconciled with impl, one source of truth

### Security: "Script isolation is complete"
**Challenge:** No resource limits specified, channels are global, storage is global
**Counter-Evidence:** `/fork` creates unbounded contexts, channels have no depth limit
**If Wrong:** Malicious or buggy scripts can DoS the runtime
**Action:** Relax — Add optional resource limits with sensible defaults

---

## Role-Specific Edge Cases & Failures

### Story Authors: Division Result Type
**Trigger:** `5 / 2` in expression — is result 2 (integer floor) or 2.5 (double)?
**Role Experience:** Unexpected math results in conditionals
**Recovery:** Possible (use explicit `5.0 / 2`)
**Mitigation:** Spec says integer division floors; add warning for implicit integer division?

### Runtime Developers: Concurrent Channel Access
**Trigger:** Multiple contexts push/pull same channel simultaneously
**Role Experience:** Race conditions, message ordering issues
**Recovery:** Difficult — requires runtime redesign
**Mitigation:** Spec says "concurrent safe" but doesn't define ordering guarantees

### Security: Persistent Storage Size
**Trigger:** `/write` with unbounded data
**Role Experience:** Disk exhaustion, storage corruption
**Recovery:** Difficult — requires storage redesign
**Mitigation:** Add storage quotas per store, per variable

---

## What's Hidden (Per Role)

**Omissions per role:**
- **Story Authors:** No guidance on safe patterns for user input handling
- **Runtime Developers:** No performance/complexity requirements for algorithms
- **Security:** No threat model, no security considerations section
- **Integrators:** No conformance test suite mentioned

**Tradeoffs per role:**
- **Story Authors:** Flexibility over safety (dynamic typing, loose validation)
- **Runtime Developers:** Simplicity over completeness (many "implementation-defined" areas)
- **Security:** Feature richness over sandboxing (channels, storage are global)

---

## Scale & Stress (Role Impact)

**At 10x contexts:**
- **Story Authors:** Story scope management becomes complex
- **Runtime Developers:** Context scheduling overhead, channel contention
- **Security:** Resource consumption grows linearly (no limits)

**At 100x contexts:**
- **Story Authors:** Debugging becomes impossible without tracing
- **Runtime Developers:** Need sophisticated scheduler, possible deadlocks
- **Security:** Trivial DoS via fork bomb

---

## Remediation

### Must Fix (Before Production)

- **Macro syntax reconciliation** (affects: Runtime Developers, Integrators) → Decide spec or impl syntax → Audit both documents
- **Resource limits specification** (affects: Security, Operators) → Add RuntimeConfig with maxContexts, maxChannelDepth, maxCollectionSize → Test limits
- **Channel generation ID documentation** (affects: Runtime Developers) → Add to spec §Channels → Update impl to match

### Should Fix (Before Production)

- **Checkpoint contract validation timing** (affects: Story Authors, Runtime Developers) → Clarify spec → Add test cases
- **Defer error handling** (affects: Story Authors) → Document behavior when defer fails → Add examples
- **Expression injection patterns** (affects: Security) → Add safe pattern documentation → Consider `[raw]` attribute

### Monitor

- **Type coercion edge cases** (affects: Story Authors) → Track bug reports → Add comprehensive test suite
- **Spec/impl drift** (affects: all) → Regular reconciliation audits → Single source of truth

---

## Final Assessment

**Soundness:** Fixable — Core design is solid, issues are mostly documentation/edge cases
**Risk:** Medium — Resource exhaustion and concurrency issues need attention
**Readiness:** Needs Work — Reconcile spec/impl, add resource limits, clarify edge cases

**Per-Role Readiness:**
- **Story Authors:** Ready with Caveats — Documentation gaps, scope confusion possible
- **Runtime Developers:** Needs Work — Spec/impl conflicts must be resolved
- **Security:** Not Ready — No resource limits, no threat model
- **Integrators:** Needs Work — Too many implementation-defined behaviors

**Conditions for Approval:**
- [ ] Macro syntax reconciled between spec.md and impl/03_preprocessor.md
- [ ] Resource limits added to RuntimeConfig (at minimum maxContexts)
- [ ] Channel generation IDs documented in spec
- [ ] Checkpoint contract validation timing clarified

**No-Go If:**
- [ ] Macro syntax remains conflicting (impacts: all runtimes will diverge)
- [ ] No resource limits exist (impacts: any production deployment)

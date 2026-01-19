# Discussion: Why not break and continue?

- Fact: In ZOH, verbs can be passed as arguments to other verbs.
- Fact: It's up to the verb how to handle its arguments. Verbs arguments are not necessarily executed.
- Fact: Loops are also verbs.

Topic: Should ZOH have break and continue?

---

## Analysis

### The Semantic Challenge

In traditional languages, `break` and `continue` are control flow statements that operate on the *syntactically enclosing* loop:

```javascript
for (let i = 0; i < 10; i++) {
    if (i == 5) break;  // Clearly refers to the for-loop
}
```

In ZOH, **loops are verbs that receive verb arguments**. The body isn't "enclosed" in the traditional sense—it's *passed* as a parameter:

```zoh
/while *condition, /verb;;
/foreach *list, *it, /verb *it;;
```

If we introduced `/break`, what would it mean?

```zoh
/while *condition, /if *x, /break;;;
:: Which verb does /break target? The /if? The /while?
```

The problem: when `/break` executes, it's running inside `/if`'s execution context, not directly inside `/while`. The `/if` verb has no idea it's being called from within a loop—it just sees `/break` as one of its verb parameters.

### The `breakif` Solution

The spec already addresses this with the `breakif` named parameter:

```zoh
/foreach *list, *it, breakif: `*it == 5`, /process *it;;
/sequence breakif: *should_stop, /verb1;, /verb2;, /verb3;;
```

This approach is:
- **Explicit**: The loop verb itself checks the condition
- **Composable**: Works regardless of nesting depth
- **Predictable**: No ambiguity about which construct is affected

### Why `/break` Would Be Problematic

1. **Scope Ambiguity**: Which enclosing verb should break? The immediate verb parent? The nearest loop verb? What if verbs are passed through multiple layers?

2. **Driver Complexity**: Every verb that accepts verb parameters would need to handle "break signals" propagating up from child verb executions.

3. **Against Philosophy**: ZOH emphasizes that verb arguments are data—verbs don't know how callers will use them. A `break` would require verbs to communicate backward with their callers.

4. **Concurrency Concerns**: With `/fork` and parallel contexts, a break signal would add complexity to already-complex concurrent execution.

### Alternative Patterns

For early termination without `break`:

```zoh
:: Using breakif
/foreach *list, *it, breakif: *found, /sequence
    /if *it, is: *target, *found <- true;;
;;

:: Using /exit for story-level termination
/if *critical_failure, /exit;;

:: Using /jump to escape complex flows
/if *done, /jump ?, "next_section";;

:: Using conditional sequencing
/sequence breakif: *done,
    /process_a;,
    /if *a_success, *done <- true;,
    /process_b;,
    /if *b_success, *done <- true;
;;
```

### Conclusion

**Recommendation: No dedicated `/break` or `/continue` verbs.**

The `breakif` pattern is more aligned with ZOH's verb-centric design, avoids control flow ambiguity, and handles the common use cases. For more complex scenarios, `/jump` to labels provides explicit flow control.

If early-exit becomes a frequent pain point, consider:
- Extending `breakif` support to more loop verbs
- Adding a `continueif` parameter with similar semantics


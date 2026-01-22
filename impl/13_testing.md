# Document 13: Testing Scenario Examples

This document outlines integration test scenarios that span multiple components of the ZOH implementation. Individual component testing is covered in each component's document; this focuses on cross-component interactions and end-to-end scenarios.

---

## Test Categories

| Category | Description | Components Involved |
|----------|-------------|---------------------|
| Compilation Pipeline | Preprocessing → Lexing → Parsing → Validation | 01-03, 12 |
| Expression Integration | Expression evaluation within verb execution | 04, 06 |
| Control Flow + Verbs | Control flow verbs with core verb combinations | 06, 07 |
| Concurrency + Channels | Multi-context scenarios with channel communication | 08 |
| Runtime + Storage | Persistent state across sessions | 09, 11 |
| Full Story Execution | Complete story with all feature types | All |

---

## 1. Compilation Pipeline Tests

### 1.1 Preprocessor → Parser Integration

```zoh
:: Test: Embedded file parsing
#embed "header.zoh"

@main
    /converse "Hello from main";
```

**Expected**: Embedded file content appears before @main in compiled story.

### 1.2 Error Location Preservation

```zoh
#embed "file_with_error.zoh"
```

**Expected**: Errors in embedded files report original file location, not expanded location.

### 1.3 Circular Dependency Detection

```zoh
:: a.zoh
#embed "b.zoh"

:: b.zoh  
#embed "a.zoh"
```

**Expected**: Fatal error: "Circular embed detected: a.zoh → b.zoh → a.zoh"

---

## 2. Expression Integration Tests

### 2.1 Expression in Verb Parameters

```zoh
*x <- 5;
*y <- 10;
$`*x + *y * 2`; -> *result;
```

**Expected**: Expression evaluated, `*result` equals 25.

### 2.2 Nested Expression Evaluation

```zoh
*list <- [1, 2, 3, 4, 5];
/count *list; -> *c;
/get "list", `*c - 2`; -> *v;  :: Get element at index 3
```

**Expected**: `*c` is 5, `*v` is 4.

### 2.3 Expression Type Coercion in Conditions

```zoh
*count <- 0;
/if `*count`, /converse "Has items";;
/if `*count + 1`, /converse "Incremented";;
```

**Expected**: First block skipped (0 is falsy), second block executed (1 is truthy).

### 2.4 String Interpolation with Expressions

```zoh
*name <- "Alice";
*score <- 100;
$`*score * 2`; -> *doubled;
$"Player ${*name} scored ${*doubled} points!"; -> *msg;
```

**Expected**: `*msg` is "Player Alice scored 200 points!"

---

## 3. Control Flow + Core Verbs Tests

### 3.1 Loop with Counter

```zoh
*sum <- 0;
*i <- 0;
/loop 5, /sequence/
    /increase "sum", *i;
    /increase "i", 1;
/;;
```

**Expected**: `*sum` equals 0+1+2+3+4 = 10.

### 3.2 While with Condition

```zoh
*counter <- 0;
/while `*counter < 5`, /increase "counter", 1;;
```

**Expected**: `*counter` equals 5.

### 3.3 Foreach over List

```zoh
*items <- [10, 20, 30];
*sum <- 0;
/foreach *items, *item, /increase "sum", *item;;
```

**Expected**: `*sum` equals 60.

### 3.4 Switch with Default

```zoh
*x <- "medium";
/switch *x, "small", 1, "medium", 2, "large", 3, 0; -> *size;
```

**Expected**: `*size` equals 2.

### 3.5 First (First Non-nothing Value)

```zoh
*a;
*b <- 42;
*c <- "hello";
/first *a, *b, *c; -> *result;
```

**Expected**: `*result` is 42 (first value where `/any` returns true).

---

## 4. Concurrency + Channels Tests

### 4.1 Producer-Consumer Pattern

```zoh
====+ @consumer;
====> @producer;

@consumer
    /loop 5, /sequence/
        /pull <data>, timeout: 2.0; -> *value;
        /converse [by: "Consumer"] *value;
    /;;

@producer
    *i <- 0;
    /loop 5, /sequence/
        /push <data>, `*i * 10`;
        /increase "i", 1;
        /sleep 0.1;
    /;;
```

**Expected**: Consumer receives 0, 10, 20, 30, 40 in order.

### 4.2 Channel Timeout with Diagnostic

```zoh
/pull <empty_channel>, timeout: 0.5; -> *result;
/diagnose; -> *diag;
```

**Expected**: `*result` is a nothing, `*diag` contains info-level timeout diagnostic.

### 4.3 Fork with Variable Transfer

```zoh
*shared <- 100;
====+ @worker *shared;
/sleep 1.0;
/converse [by: "Main"] *shared;

@worker
    /increase "shared", 50;
    /converse [by: "Worker"] *shared;
```

**Expected**: Forked context has its own copy; main still shows 100, worker shows 150.

### 4.4 Call with Return Value

```zoh
*x <- 5;
<===+ @double *x; -> *result;

@double
    /increase "x", *x;
    /exit *x;
```

**Expected**: `*result` is 10, returned via `/exit`.

### 4.5 Signal and Wait Coordination

```zoh
====+ @waiter;
/sleep 0.1;
/signal "ready", true;
/converse "Signal sent";

@waiter
    /wait "ready", timeout: 5.0;
    /converse "Received signal";
```

**Expected**: Waiter resumes after signal; both messages appear.

---

## 5. Runtime + Storage Tests

### 5.1 Persistent Variable Lifecycle

```zoh
:: Session 1
*score <- 0;
/increase "score", 100;
/write *score;

:: Session 2 (new runtime)
/read *score;
<- *score;
```

**Expected**: Value persists across runtime sessions; `*score` is 100.

### 5.2 Storage Nested Structures

```zoh
*data <- {
    "nested": {"a": {"b": "deep"}}
};
/write *data;

:: Later or in new session:
/read *data;
/get "data", "nested"; -> *n;
/get "n", "a"; -> *a;
/get "a", "b"; -> *deep;
```

**Expected**: After read, `*deep` is "deep".

---

## 7. Edge Case Tests

### 7.1 Empty/Null Handling

```zoh
*empty;
*emptyList <- [];
*emptyMap <- {};

/if *empty, /converse "Should not print";;
/foreach *emptyList, *item, /converse "Should not print";;
/get "emptyMap", "missing"; -> *value;
```

**Expected**: No converse output; `*value` is a nothing.

### 7.2 Deeply Nested Structures

```zoh
*deep <- {
    "l1": {
        "l2": {
            "l3": [1, 2, 3]
        }
    }
};
/get "deep", "l1"; -> *l1;
/get "l1", "l2"; -> *l2;
/get "l2", "l3"; -> *l3;
/get "l3", 2; -> *val;
```

**Expected**: `*val` is 3.

### 7.3 Unicode Handling

```zoh
*greeting <- "こんにちは世界 🌍";
/converse *greeting;
*name <- "José García";
$"Welcome, ${*name}!"; -> *msg;
/if `*msg == "Welcome, José García!"`, /converse "Unicode OK";, else: /fatal "Unicode failed";;  
```

**Expected**: Full Unicode support in strings and interpolation.

### 7.4 Recursion (via Call)

```zoh
*depth <- 0;
<===+ [inline] @recursive *depth; -> *finalDepth;
/converse *depth;

@recursive
    /increase "depth", 1;
    /if `*depth < 100`, <===+ [inline] @recursive *depth;;;
    /exit *depth;
```

**Expected**: `*depth` is 100 (via inline), or hits stack limit with clear error.

---

## 8. Error Scenario Tests

### 8.1 Type Mismatch Diagnostics

```zoh
*text <- "hello";
/increase "text", 5;
```

**Expected**: Fatal diagnostic with code "invalid_type".

### 8.2 Missing Required Parameter

```zoh
/set;
```

**Expected**: Fatal diagnostic at parse time or validation.

### 8.3 Invalid Attribute

```zoh
/converse [nonexistent:true] "Hello";
```

**Expected**: Warning diagnostic for unknown attribute.

### 8.4 Channel Already Closed

```zoh
/close <ch>;
/push <ch>, "value";
```

**Expected**: Error diagnostic with code "closed".

---

## Test Execution Guidelines

### Priority Order

1. **P0 - Critical**: Compilation pipeline, fatal error handling
2. **P1 - High**: Core verb execution, expression evaluation  
3. **P2 - Medium**: Control flow, concurrency basics
4. **P3 - Low**: Edge cases, performance limits

### Assertion Patterns

```
// Pseudocode for test assertions
assertValue(context, "varName", expectedValue)
assertDiagnostic(result, level, code)
assertNoFatalDiagnostics(result)
assertContextComplete(context)
assertChannelEmpty(channel)
```

---

## Related Documents

- [01_lexer.md](01_lexer.md) - Token-level tests
- [02_parser.md](02_parser.md) - AST structure tests
- [06_core_verbs.md](06_core_verbs.md) - Individual verb tests
- [12_validation.md](12_validation.md) - Validation rule tests

# Runtime Design

Runtime should be layered in the following manner: Runtime, Context, Story.

A runtime may have multiple contexts while each context may only be at a certain position of a story.

## Variable Resolution

The runtime should lookup for variables in story scope first then context scope.

In case of name conflicts, context variables are shadowed by story variables.

It is recommended to be verbose and explicit when naming variables.

## Syntactic Sugar

Verbs may have syntactic sugar forms, but these only exist in the source code and should be translated to the standard form by pre-processors.

## Handlers

A runtime should be designed in a modular fashion that it can be extended with the following:

### Pre-processor
Pre-processors are called before the runtime parser and can take action against the raw story data.
They are provided with the story name, the metadata entries and the story body text, and can temper them in anyway.
Pre-processors can invalidate the story by returning a fatal diagnostic.

#### Standard Preprocessors
- TODO

### Compiler
Compilers are called to verify and convert the story to runtime data structures.
Compilers work by repeatedly asking if any registered compiler is interested in the current node. An interested compiler may read ahead and perform its logics to generate runtime data structures or modify already generated ones, then the next compiler is asked, and so on, until no compiler is interested in the current node. The runtime should move ahead and repeat the process until the end of the story. Noted that once the runtime moves on, all compiled elements up to the point should be finalized.
In other words, because compilers are called in order of priority, compilers with lower priority can modify results from compilers with higher priority.
Compiled elements should track their source line number and story body line number in the story, the difference is story body line number is based on actual parsed story body.

#### Standard Compilers
- TODO

### Story Validator
Story validators are called before running the story. Any story validator can output diagnostics and prevent the story from running.

### Verb Validator
Verb validators are called by a standard story validator to verify the compiled verb calls are valid after compilers are done.

### Verb Driver
Verb drivers are called by the runtime/context to perform the verb call at runtime.

## Handler Priority

Handlers should be registered with priority value so the runtime can call them in order.

For verb drivers, the runtime only calls the one with hightest priority, if multiple verb drivers map to the same verb name.

The highest priorty is -2^31 and the lowest is 2^31-1.

## Error Handling

Handlers can return diagnostics to the runtime. Diagnostics are leveled in severity of "info", "warning", "error" and "fatal". Fatal diagnostics immediately terminate the context, while other diagnostics should be logged and reported.

Contexts should keep a list of diagnostics, which may be used by the runtime, or other contexts in case it is a forked context.

## State Management
Handlers should be stateless and concurrent safe. State should only exists in runtime and context.

## Asset Management
Asset paths should be abstracted away and assets should be identified by "address". Addresses can be file paths but not limited to.

## Channel Management

Noted that closing a channel should instantly notify all pullers. No puller should be mistakenly pull from a new channel with the same name.
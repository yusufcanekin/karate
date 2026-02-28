# Requirements & Analysis

## Problem Statement
Karate supports executing feature files that may invoke Java methods, which in turn can trigger additional (nested) Karate feature executions. During such nested execution flows, the JavaScript evaluation environment of the outer scenario is expected to remain consistent.

However, as demonstrated in issue #2502, when a Karate feature calls a Java method that executes another Karate feature (runner-within-runner scenario), subsequent JavaScript function evaluations in the outer scenario may fail with a `ReferenceError`, even though the referenced variable still exists in Karate's internal variable map.

This indicates that nested feature execution can disrupt or replace the JavaScript execution context or its bindings, causing previously accessible variables to become unavailable to JavaScript functions.

The defect occurs specifically when:
1. A variable is defined in a Karate scenario.
2. A JavaScript function references that variable.
3. A nested feature execution occurs (via Java).
4. The same JavaScript function is invoked again and fails due to the variable no longer being present in the JS bindings.

### Expected Behavior
* Variables defined during scenario execution must be accessible from subsequent JavaScript evaluations within the same scenario.
* Nested feature execution (whether initiated by `call read(...)` or by Java invoking Karate while an outer scenario is running) must preserve the outer scenario's JavaScript context and bindings.
* After returning from nested execution, JavaScript evaluation in the outer scenario must see the same variable scope as before the nested call.

### Observed Behavior (issue #2502 reproducer)
* After nested feature executiom, Karate still prints the varaible value from its variable map, but JavaScript evaluation fails with `ReferenceError: "<var>" is not defined`.
* This indicates that Karate's scenario variable map remains intact, but the JavaScript engine bindings/context used for evaluation no longer contains the variable.

### Scope of Impact
This issue affects any scenario that:
- Defines variables accessed by JavaScript functions, and
- Performs nested feature execution (directly or indirectly via Java).

Because many existing tests rely on JavaScript evaluation and nested feature calls, the regression risk of modifying JS context handling is high.
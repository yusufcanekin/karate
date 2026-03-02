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

## Requirements
* **REQ-1:** A JavaScript function evaluate within a scenario must be able to access variables defined earlier in that scenario (and any configured global JS bindings).
* **REQ-2:** Executing a nested feature via Java (runner within runner) must not break the JS context of the outer scenario.
* **REQ-3** (regression safety): Fix must not change behavior of existing scenarios that do not use nested execution (including standard call chains and config loading).

## Requirement Table
| Req ID   | Requirement                                                                            | Why it matters                                 | Impacted mechanisms                           | Example areas                     | Verification              |
| -------- | -------------------------------------------------------------------------------------- | ---------------------------------------------- | --------------------------------------------- | --------------------------------- | ------------------------- |
| REQ-JS-1 | JS functions must access variables defined earlier in scenario and configured globals. | Core JS integration behavior.                  | ScenarioEngine bindings, JS context injection | js-read.*, js-call.*              | Existing JS tests pass    |
| REQ-JS-2 | Java-triggered nested execution must restore parent ScenarioEngine.                    | Prevents context corruption (#2502).           | Runner.runFeature, Suite, ScenarioRuntime     | JsVarAccess.feature               | KarateJsGlobalTest passes |
| REQ-JS-3 | callSingle functions must remain correctly bound to runtime context.                   | callSingle caching depends on stable bindings. | callSingleCache, thread-local engine          | parallel/call-single-*            | callSingle tests pass     |
| REQ-JS-4 | callonce shared-scope variables/functions must remain visible and not leak.            | Shared-scope include is common pattern.        | callOnceCache, scope merge                    | callonce-global.*, common.feature | callonce tests pass       |
| REQ-JS-5 | Fix must not break non-nested execution behavior.                                      | Regression safety.                             | Full engine lifecycle                         | Entire test suite                 | `mvn test` passes         |


## Identified Scenarios
### Scenarios that execute JavaScript or reference JS sources

### Confirmed scenarios relying on JS global bindings

The following scenarios rely on Karate variables being available as global identifiers inside JavaScript functions:
Under jscall2:
call.feature
call-once.feature
call-single.feature
local.feature (jscall2)

local.feature - JS global + nested on the (* callonce read(...))
all.feature - likely JS globals --> definitely JS global
parallel-outline-simple.feature - nested 
parallel.feature - likely --> 

### Nested execution entry points (Java vs feature calls)

Nested Karate execution can occur in two ways:

1. **Feature-to-feature nesting**, triggered from `.feature` files using `call read(...)`, `callonce`, and `callSingle`. These flows can create parent→child→grandchild chains and rely on suite-level caches for `callonce`/`callSingle`.

2. **Java-triggered nesting (runner-within-runner)**, where Java code invokes Karate execution while an outer scenario is already running. The `Runner.runFeature(...)` API constructs a new `Suite` and `FeatureRuntime` and runs the feature, which may create a new JavaScript execution context and must not disrupt the outer scenario's JS bindings.

We identified that the main JS environment/context was being created in Runner.java. After analysing the file we found that the main entry points for nested calls were associated with the variables: callOnceCache and callSingleCache

Example .feature files with nested calls:
/com/intuit/karate/core/jscall
js-call.feature
js-called.feature
js-callonce.feature

core/
call-response.feature - nested 
_mock.feature - nested

..
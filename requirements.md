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
| Impact ID | Execution / nesting mechanism                                                            | Representative locations (examples)                                                                                                                  | Expected behavior (intended lifecycle)                                                                                                                                                                                                    | Observed / risk behavior                                                                                                                                                                                                                                                                                      | Affected area(s)                                                                                          | Severity      | Notes / how to verify                                                                                                                                                                                             |
| --------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| IA-01     | **Java-triggered nested execution via `Runner.runFeature()`** (runner-within-runner)     | Any feature / JS that calls `Java.type('com.intuit.karate.Runner')` then `Runner.runFeature(...)`; Java tests/docs that use `Runner.runFeature(...)` | After nested execution completes, the **parent feature’s JS bindings / functions must still resolve variables** from the parent runtime context. The nested run must not “orphan” the parent JS context.                                  | Investigation shows `Runner.runFeature()` creates `new Suite()` and the nested call can become a **new root** (caller seen as none), leading to a **new `ScenarioEngine` with an empty vars map** and a fresh JS engine context. This can cause JS functions in the parent to lose access to variables.  | `Runner.runFeature`, `Suite`, `FeatureRuntime`, `ScenarioRuntime`, `ScenarioEngine` / JS engine lifecycle | **High**      | Verify with a scenario where a JS function references a variable defined before the nested call, and fails after `Runner.runFeature`.      |
| IA-02     | **Feature-level nested call via `call read('child.feature')`** (normal nested execution) | Features that use `* call read('...feature')` chains                                                       | Caller and callee should execute with well-defined scoping rules: **isolated by default**, unless explicitly using shared scope patterns. Caller’s JS context should remain consistent after child returns.                               | Lower risk than IA-01 because this call path is part of the normal engine flow; still relevant if implementation accidentally swaps the active engine.                                                                                                                                              | `ScenarioRuntime` call stack handling, variable scope rules, call framing                                 | Medium        | Verify that after returning from `call read`, parent JS functions still resolve caller variables. Also check that callee does not unexpectedly mutate parent scope unless shared scope is intended.               |
| IA-03     | **Shared-scope include pattern** via `callonce read('common.feature')` in `Background`   | Features that intentionally use `callonce` as an “include” to contribute vars/config (e.g., “common setup” features, headers/token setup)            | The called feature can **contribute variables and config** (headers/cookies/etc.) into the caller’s shared scope, and these should remain available for subsequent scenarios unless overwritten.                                          | If JS engine / thread-local engine is disturbed by nested execution, **injected config vars** (e.g., token, headers) or helper functions may become unavailable or bound to the wrong context.                                                                                                                | Shared scope merge rules; config propagation into runtime; JS function bindings                           | **High**      | Verify that variables/config injected in Background still work across multiple scenarios in the same feature. Also verify “reset/override” operations like `configure headers = null`.                            |
| IA-04     | **`callonce` caching / lifecycle** (run once, reuse across scenarios)                    | Features that use `* callonce read('...feature')`, especially in Background                                                                          | `callonce` should run once per intended scope (suite/feature), and **must not leak scenario-local variables across scenarios**. Also, helper functions defined via `callonce` should remain callable and correctly bound when used later. | If lifecycle handling is wrong --> either: (a) **leakage** across scenarios, or (b) helper functions become unbound / wrong engine after a context switch.                                                                                                                                           | `callOnceCache`, shared scope rules, JS engine binding stability                                          | High          | Verify both: (1) “no leak” (scenario-defined vars don’t appear in later scenarios), and (2) helper functions defined in `callonce` can access runtime vars (like `responseStatus`) correctly when executed later. |
| IA-05     | **`callSingle` caching (suite/global-like)** from config or from feature                 | `karate.callSingle('...feature', ...)` in `karate-config.js` and `* def x = karate.callSingle('...feature')` in features                             | `callSingle` should run once per intended global scope and return cached results consistently. **Functions/vars produced by callSingle** should remain callable and resolve runtime variables correctly when used inside scenarios.       | Since `callSingle` relies on caching + runtime context interplay, engine swaps can cause functions to be attached to the wrong polyglot context, or lose visibility of variables (similar shape to the reported failure).                                                                                     | `callSingleCache`, config initialization, JS function/object binding across calls                         | **Very High** | Verify by calling the returned function(s) in multiple scenarios and ensuring they can read runtime variables (e.g., `responseStatus`) correctly. Also verify access via `__arg` if used.                         |
| IA-06     | **Parallel execution + callonce/callSingle**                                             | Features under `parallel/` that combine `parallel`, `callonce`, `callSingle`                                                                         | Parallel runs must preserve isolation between threads while still honoring intended caches (`callonce`, `callSingle`) and shared scope rules. JS context must not “bleed” across threads.                                                 | Thread-local engine management is sensitive here: if engine restore/set is wrong, cross-thread context corruption or intermittent failures possible.                                                                                                                                                       | Thread-local `ScenarioEngine`, suite-level caches, parallel runner                                        | **High**      | Run parallel suite multiple times; look for missing functions/vars or inconsistent cached results.                                                                                                    |
| IA-07     | **Config lifecycle toggles** (`evalKarateConfig` true/false)                             | Java API `Runner.runFeature(..., evalKarateConfig)` usage                                                                                            | If config evaluation is disabled, features should behave predictably (no config vars); if enabled, config vars/functions should be available. **Nested execution should not corrupt the caller’s engine**.                    | When `Runner.runFeature` creates a new suite and config behavior differs, it can create confusion about what “globals” should exist.                                                                                                     | `Runner.runFeature`, config loading, caller/callee boundary                                               | Medium–High   | Verify both modes with a feature that expects config-provided functions/vars. Ensure returning from nested run does not change caller behavior.                                                                   |
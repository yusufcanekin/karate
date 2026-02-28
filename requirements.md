# Requirements & Analysis

## Behavior
### Expected Behavior
* Variables defined during scenario execution must be accessible from subsequent JavaScript evaluations within the same scenario.
* Nested feature execution (whether initiated by `call read(...)` or by Java invoking Karate while an outer scenario is running) must preserve the outer scenario's JavaScript context and bindings.
* After returning from nested execution, JavaScript evaluation in the outer scenario must see the same variable scope as before the nested call.

### Observed Behavior (issue #2502 reproducer)
* After nested feature executiom, Karate still prints the varaible value from its variable map, but JavaScript evaluation fails with `ReferenceError: "<var>" is not defined`.
* This indicates that Karate's scenario variable map remains intact, but the JavaScript engine bindings/context used for evaluation no longer contains the variable.

## Requirements
* REQ-1: A JavaScript function evaluate within a scenario must be able to access variables defined earlier in that scenario (and any configured global JS bindings).
* REQ-2: Executing a nested feature via Java (runner within runner) must not break the JS context of the outer scenario.
* REQ-3 (regression): Fix must not change behavior of existing scenarios that do not use nested execution (including standard call chains and config loading).

## Requirement Table
| Req ID | Requirement | Why it matters | Impacted mechanisms | Example features | Verification|
|--------|------------|---------------|--------------------|------------------|--------------|
| REQ-1 | JS eval in a scenario must access variables defined earlier in the scenario (and configured JS globals). | Many features rely on JS helpers reading previously defined vars. | JS engine bindings, variable injection into JS context | js-read.feature, js-arrays.feature | Existing JS tests pass |
| REQ-2 | Nested feature execution must not invalidate the outer scenario’s JS context. | Issue #2502 shows JS loses access to vars after nested execution. | Nested runtime lifecycle, context creation/restoration | JsVarAccess.feature (@JsScenarioWithCallToNestedRunner) | KarateJsGlobalTest (currently failing) |
| REQ-3 | Fix must not change behavior of scenarios without nested execution (regression safety). | Context fixes can break existing tests. | Runner/Suite flow, config loading | js-read.feature, call-feature.feature | Full `mvn test` passes |
## Identified Scenarios that execute JavaScript or reference JS sources
# AGENTS.md — Enterprise Baseline (Project-Agnostic)

## Authority
thisFile IS compactRepresentation of project-agnostic-AGENTS-readable.md
project-agnostic-AGENTS-readable.md IS canonicalSource
thisFile MUST_NOT weaken|contradict project-agnostic-AGENTS-readable.md
shorterInstructionFile MUST_NOT weaken|contradict project-agnostic-AGENTS-readable.md
condensedFile MUST_RETAIN architectureBoundaries
condensedFile MUST_RETAIN asyncContracts
condensedFile MUST_RETAIN errorClassification
condensedFile MUST_RETAIN circuitBreaker
condensedFile MUST_RETAIN security
condensedFile MUST_RETAIN rationale
condensedFile MUST_RETAIN testGates

## Architecture
transport MUST_BE thin ; parse+call+map+return ONLY
businessRules MUST_RESIDE_IN services|domain ; NOT controllers|templates|dataAccess
dataAccess MUST_BE_BEHIND repository|dataModule
externalIntegrations MUST_BE_BEHIND adapterInterface
construction PREFER composition|factory OVER deepInheritance

## Performance
hotPath MUST_NOT use blockingSyncAPIs|unboundedLoops|expensiveSerializationPerRequest
hotPath MUST_USE asyncAPIs|boundedWorkUnits|streaming|workers|inputSizeGuardrails

## Async and Error Contracts
publicAPIs SHOULD_BE consistently async
errorSurface MUST_USE one deterministic path ; NOT multiple
errorMapping MUST_BE centralized

## Error Classification
transient CLASSIFY_AS networkTimeout|502|503|rateLimit → retry
permanent CLASSIFY_AS 400validation|404|schemaMismatch → failImmediate ; noRetry
upstream CLASSIFY_AS thirdPartyOutage|queueUnavailable → circuitBreaker
internal CLASSIFY_AS nullRef|assertion|invariant → failFast,alert,noRetry
error MUST_CARRY classification ; callers branch on category NOT stringMatch|statusCode

## Circuit Breaker
externalDependency MUST_TRACK failureCount|errorRateWindow
circuit OPEN_WHEN failureThreshold breached ; reject immediately
circuit HALF_OPEN_AFTER cooldownPeriod ; allow singleProbe
circuit CLOSE_WHEN probe succeeds
circuitTransition MUST_LOG dependencyName,threshold,window

## Retry
retry ONLY_FOR transient errors
retry MUST_USE exponentialBackoff+jitter ; NOT fixedInterval|immediateUnderLoad
retry MAX 3 on syncPaths ; configurable on asyncPaths
retry REQUIRES idempotentOperation
retry MUST_LOG attemptNumber,delay,errorCategory
retry.exhausted MUST surface clearTerminalError ; NOT swallow

## Fallback
dependency.unavailable PREFER degradedResponse(cached|default|featureOff) OVER hardFailure
degradation MUST_LOG ; NOT silent

## Security
untrustedInput MUST validate+normalize at boundaries
enumFields MUST_USE allowlists ; strictSchemaValidation for payloads
dbOperations MUST_USE parameterized|approvedQueryLayers
clientErrors MUST_NOT leak internals
secrets MUST_NOT appear in logs|source ; use envVar|secretManager
serviceCredentials MUST_APPLY leastPrivilege

## Logging
logs MUST_BE structured ; NOT freeFormStrings on hotPaths
verbosity MUST_BE gated by level
logs MUST_INCLUDE correlationId,latency,statusOutcome
metrics EMIT throughput,errorRate,latency,saturation

## Code Comments
comments DEFAULT rationale(WHY) ; NOT mechanics(what)
nonTrivialBlocks REQUIRE WHY comment
format: `WHY:<rationale> / TRADEOFF:<downside> / VERIFY_IF_CHANGED:<retest>`

## Docs Sync
publicBehaviorChange REQUIRES README+devDocs+runbooks+changelog updated
exampleSnippets MUST_EXECUTE against current code

## Testing Gates
merge REQUIRES lintPass,unitTests(coreLogic+errorPaths),integrationTests(boundary+criticalWorkflows)
concurrencySensitiveChange REQUIRES stress|race|orderingValidation
skippedCheck MUST_DOCUMENT why,risk,followUpOwner+date

## Review Checklist
1. noBlockingSyncAPIs in hotPaths
2. noAsyncConsistencyRegressions
3. errorHandling deterministic+policyAligned
4. errorsClassified(transient|permanent|upstream|internal)
5. externalCalls use circuitBreaker OR documented exempt
6. retries idempotent+bounded+backoffWithJitter
7. securityValidation at boundaries
8. logging useful+structured+notThroughputDominant
9. docs+tests updated with behaviorChanges
10. rationaleComments where nonObvious

## Exception Process
exception REQUIRES justification: why,whySaferAltsNotUsed,scope+blastRadius,verificationPerformed+rollbackPlan

## Agent Delivery Contract
1. summarize changes+why
2. report validationCommands+outcomes
3. callOut knownLimitations|followUpWork
4. state constraints ; NO hiddenAssumptions

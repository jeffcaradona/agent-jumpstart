# AGENTS.md — AI Agent Service

agentServiceFile EXTENDS project-agnostic-AGENTS-rdf.md
agentServiceFile WINS_ON conflict

## Authority
thisFile IS compactRepresentation of agnostic-agent-service-AGENTS-readable.md
agnostic-agent-service-AGENTS-readable.md IS canonicalSource
thisFile MUST_NOT weaken|contradict agnostic-agent-service-AGENTS-readable.md
shorterInstructionFile MUST_NOT weaken|contradict agnostic-agent-service-AGENTS-readable.md
condensedFile MUST_RETAIN agentLoopSafety
condensedFile MUST_RETAIN toolContracts
condensedFile MUST_RETAIN groundingRules
condensedFile MUST_RETAIN inferenceResilience
condensedFile MUST_RETAIN sessionLifecycle
condensedFile MUST_RETAIN promptManagement
condensedFile MUST_RETAIN errorDifferentiation
condensedFile MUST_RETAIN security
condensedFile MUST_RETAIN rationale

## Architecture
entryPoint(index.js) HANDLES acceptInput,startSession,callAgent,returnResult ; NO businessLogic
agentLoop(agent.js) OWNS generateText,toolRegistry,maxStepsPolicy ; NO toolImplementation
tools(tools/*.js) ONE_FILE_PER domain(nodes,vms,storage) ; calls apiClient ; NO directHTTP
apiClient(client/api.js) OWNS allHTTP ; applies sessionIdHeader,envelopeUnwrapping,errorClassification ; NO inferenceLogic
config(config.js) ALL envBackedConfig ; NO unsafeProductionDefaults

## Agent Loop Safety
generateText MUST_SET maxSteps explicitly ; NEVER omit|Infinity
maxSteps DEFAULT 10 ; MAX 25 without explicit sign-off (see Exception Process)
agentInvocation MUST_WRAP inTimeout ; config.agent.timeoutMs DEFAULT 30000
agentLoop MUST_BE stateless across invocations
agentLoop MUST_NOT store toolState|conversationHistory|modelContext in moduleScope
agentLoop MUST_NOT cache inferenceResults across sessions

## Tool Contract
tool MUST_DO exactly oneAction against oneEndpoint ; NO multiStep|conditional in execute
tool.description IS loadBearing ; treat as publicAPIContract
tool.description MUST_DESCRIBE returnData ; NOT just whatItDoes
tool.description MUST_INCLUDE naturalLanguageTriggerPhrases
adjacentTools MUST_BE explicitly differentiated in descriptions
tool.description MUST_NOT use vagueVerbs without preciseObjectNoun
tool.parameters MUST_BE minimal+typed
tool.parameters MUST_HAVE z.describe() annotation
tool.parameters MUST_USE z.enum() for boundedValueSets ; NOT z.string()
tool.execute MUST_RETURN apiData minimally transformed
tool.execute MUST_NOT fillMissingFields|mergeCalls|formatForModel

## Grounding
systemPrompt MUST_CONTAIN "Never answer cluster state from memory; always use tools; if no tool, say so explicitly"
agentAnswer SHOULD_LOG toolResults alongside finalAnswer for postHocVerification ; NOT inline

## Inference Resilience
inferenceCall MUST_HAVE explicitPerCallTimeout ; recommended 20000ms (Mistral 7B)
ollamaHTTP5xx|connectionRefused → upstream ; circuitBreaker,noRetryOnSyncPath
ollamaHTTP429 → transient ; backoffRetry,max2
malformedModelResponse → internal ; failFast,logRawResponse,noRetry
modelNoToolCallNoAnswer → internal ; fail(NO_MODEL_OUTPUT),noRetry
maxStepsExceeded → internal ; fail(MAX_STEPS_EXCEEDED),noRetry
ollama OPEN_CIRCUIT_AFTER 3 consecutiveFailures within 60s ; return INFERENCE_UNAVAILABLE ; NO queue|retry

## API Client
apiClient IS soleOwner of allHTTP ; NO directFetchInTools|agents
outboundRequest MUST_INCLUDE X-Agent-Session-Id
apiClient MUST_UNWRAP {success,data,error} envelope ; tools receive data|thrown
apiClient MUST_CLASSIFY errors before throwing
NODE_NOT_FOUND|VM_NOT_FOUND → permanent
INVALID_QUERY_PARAM → permanent
RATE_LIMITED → transient (honor Retry-After)
PROXMOX_UNREACHABLE|PROXMOX_TIMEOUT → upstream
INTERNAL_ERROR → upstream
networkFailure → upstream

## Session Lifecycle
sessionId MUST_USE crypto.randomUUID() ; NO externalLibrary
sessionId MUST_BE generated before anyToolCall
sessionId MUST_BE attached to allOutboundRequests via X-Agent-Session-Id
sessionId MUST_BE included in allLogEntries
sessionId MUST_BE returned to caller
sessionClose MUST_EMIT structuredSummaryLog: {ts,sessionId,prompt,stepsUsed,toolsCalled,durationMs,outcome,errorCode}
failedSession MUST_INCLUDE errorCode,errorClass in summaryLog

## Prompt Management
systemPrompt MUST_LOAD from envConfig|configFile ; NOT hardcoded in agent.js
systemPromptChange IS behavioralChange ; MUST commit+validateGoldenPaths+noteToolSelectionChanges
userInput MUST_PASS as prompt param ONLY ; NEVER interpolate into systemPrompt

## Security
fullUserPrompt MUST_NOT log at infoLevel in production ; use truncatedHash|length
toolDescriptions|paramAnnotations MUST_NOT contain networkTopology|hostnames|credentials
apiClient MUST_NOT expose rawProxmoxTokens ; use envVars only
systemPrompt MUST_NOT contain secrets|internalIPs|credentials

## Logging
debug: fullToolCallParams,rawToolResults
info: sessionStart,sessionEndSummary,toolCallNames(NOT params)
warn: stepCountApproachingMaxSteps,RetryAfterHonored,circuitHalfOpen
error: sessionFailure,circuitOpen,inferenceTimeout,unhandledToolError
debug MUST_NOT log in production by default ; gate with LOG_LEVEL=debug

## Code Comments
toolDescription.change REQUIRES comment: whyWordingChanged,whatBehaviorItCorrects
maxSteps.value REQUIRES comment: reasoningForLimit
toolResult.transformation REQUIRES comment: why,whatIsLost
systemPromptReference REQUIRES comment: version|lastValidatedDate

## Testing Gates
goldenPathQueries(5) MUST_PASS before merge on agentLoop|tools|systemPrompt|apiClient changes
toolSelectionTest REQUIRE oneTest per tool asserting modelSelectsIt given representativePrompt
runawayLoopTest MUST assert maxSteps:3+alwaysErroringTool → MAX_STEPS_EXCEEDED,noHang
envelopeErrorTest MUST assert success:false → classifiedError ; NOT null|swallowed

## Review Checklist
1. maxSteps set+withinLimits
2. sessionTimeout present+coversFullInvocation
3. noMutableSharedState in agentLoop|tools
4. each tool doesExactlyOneThing against oneEndpoint
5. toolDescriptions differentiate adjacentTools
6. toolParams use z.enum() for boundedSets
7. tools doNot transform|invent data
8. systemPrompt contains groundingInstruction
9. userInput NOT interpolated into systemPrompt
10. apiClient classifies allErrorCodes before throwing
11. sessionId generated once+present on allOutboundRequests
12. sessionSummaryLog emitted on success+failure
13. noSecrets|credentials|topology in toolDescriptions|prompts
14. goldenPathQueries validated

## Exception Process
INHERIT baseline ; additionalSignOffRequired for:
- maxSteps > 25
- bypassApiClient for directDownstreamCall from tool
- interpolateRuntimeData into systemPrompt
- disableSessionTimeout

## Agent Delivery Contract
1. state changes in agentLoop|tools|systemPrompt|apiClient and why
2. report goldenPathQueries tested+outcomes
3. report maxSteps consumed per goldenPathQuery
4. callOut toolSelectionBehaviorChanges
5. callOut promptLimitations|modelBehaviorEdgeCases
6. list deferredWork+reasons

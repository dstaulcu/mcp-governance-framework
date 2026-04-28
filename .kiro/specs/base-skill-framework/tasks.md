# Implementation Plan: BaseSkill Framework

## Overview

Implement a security-by-default Python class hierarchy for building data retrieval Skills in high-security (Intel/Defense) environments. The implementation follows a bottom-up dependency order: foundational data models and utilities first, then core framework components (audit, HWM, confidence, circuit breaker), then the security policy engine, then the security envelope enforcers, then the BaseSkill ABC (which delegates to the policy engine), then the hook adapter layer, then the LangGraph reference adapter, and finally the example skill and steering file. Each task builds on previous tasks so there is no orphaned code.

## Tasks

- [ ] 1. Set up project structure and exception hierarchy
  - [ ] 1.1 Create project directory structure and `__init__.py` files
    - Create directories: `base_skill_framework/`, `base_skill_framework/adapters/`, `tests/`
    - Create `base_skill_framework/__init__.py` with public API exports
    - Create `base_skill_framework/adapters/__init__.py`
    - Create `tests/__init__.py`
    - Add `pytest` and `hypothesis` to project dependencies
    - _Requirements: 1.1, 6.5_

  - [ ] 1.2 Implement exception hierarchy in `base_skill_framework/exceptions.py`
    - Create `BaseSkillError` root exception
    - Create all exception classes: `ClassificationMissingError`, `ClassificationInvalidError`, `ClassificationExceedsHWMError`, `ClassificationCeilingExceededError`, `ClassificationMarkCitationMismatchError`, `SourceMissingError`, `AuthenticationRequiredError`, `TotalRetrievalFailureError`, `ToolTimeoutError`, `CircuitBreakerOpenError`
    - Each exception must accept the constructor parameters defined in the design (e.g., `ClassificationMissingError(source, tool_name)`)
    - _Requirements: 2.2, 2.4, 5.6, 11.2, 12.3, 13.2, 15.2, 17.5, 19.2, 19.4_

- [ ] 2. Implement Classification Model
  - [ ] 2.1 Implement `ClassificationLevel` enum and utilities in `base_skill_framework/classification.py`
    - Define `ClassificationLevel(IntEnum)` with values U=0, C=1, S=2, TS=3, TS_SCI=4
    - Add `display` and `portion_mark` properties to each member
    - Implement `parse_classification(value: str) -> ClassificationLevel` supporting all input formats from the design's parse mapping table
    - Implement `compare_classifications(a, b) -> ClassificationLevel` returning `max(a, b)`
    - Raise `ClassificationInvalidError` for unrecognized strings
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 3.1_

  - [ ]* 2.2 Write property test for classification round-trip
    - **Property 1: Classification round-trip**
    - For any valid `ClassificationLevel`, `parse_classification(level.display) == level` and `parse_classification(level.portion_mark) == level`
    - Use `st.sampled_from(ClassificationLevel)` as generator
    - **Validates: Requirements 10.2, 10.4**

- [ ] 3. Implement Data Models
  - [ ] 3.1 Implement data models in `base_skill_framework/models.py`
    - Create `SourceMetadata` frozen dataclass with `title`, `system`, `url` fields
    - Create `UserContext` frozen dataclass with `user_id`, `auth_method`, `clearance_level` fields
    - Create `DataFragment` dataclass with `content`, `classification`, `source`, `confidence` fields (confidence defaults to 0.5)
    - Create `AuditRecord` dataclass with all fields from the design: `timestamp`, `user_id`, `auth_method`, `tool_name`, `status`, `data_classification`, `error_description`, `prev_hash`, `record_hash`, `metadata`
    - Include `HOOK_ALLOW` and `HOOK_DENY` in the set of valid status values
    - Create `HookDecision` dataclass with `allow` (bool), `reason` (str), `modified_arguments` (optional dict), `modified_result` (optional Any)
    - _Requirements: 2.1, 11.1, 12.1, 15.1, 15.5, 4.1, 21.2_

  - [ ]* 3.2 Write property test for UserContext immutability
    - **Property 12: User context immutability**
    - For any constructed `UserContext`, attempting to modify any attribute shall raise `FrozenInstanceError` or `AttributeError`
    - Use `st.text()` for random field values
    - **Validates: Requirements 15.2, 15.5**

- [ ] 4. Implement High-Water Mark Tracker and Confidence Scorer
  - [ ] 4.1 Implement HWM tracker in `base_skill_framework/hwm.py`
    - Create `HighWaterMarkTracker` class with `__init__(default)`, `register(level)`, `get_hwm()`, `reset()` methods
    - Store levels in a set; `get_hwm()` returns `max(levels)` or `default` if empty
    - _Requirements: 3.1, 3.2, 3.3, 3.5, 3.6_

  - [ ]* 4.2 Write property tests for HWM tracker
    - **Property 2: High-Water Mark is the maximum**
    - For any non-empty list of `ClassificationLevel` values, `get_hwm()` returns the maximum
    - **Property 3: High-Water Mark idempotence**
    - Registering a level ≤ current HWM does not change the HWM
    - **Validates: Requirements 3.1, 3.3, 3.5, 3.6**

  - [ ] 4.3 Implement confidence scorer in `base_skill_framework/confidence.py`
    - Create `ConfidenceScorer` class with static methods: `combine_confidence(scores, weights)`, `compare_confidence(a, b)`, `build_disclaimer(aggregate, fragments, threshold)`
    - `combine_confidence` computes weighted average bounded by `[min(scores), max(scores)]`
    - `build_disclaimer` returns `None` if aggregate ≥ threshold, else a formatted warning string
    - _Requirements: 12.4, 12.5, 12.6, 12.7, 12.8_

  - [ ]* 4.4 Write property test for confidence scoring
    - **Property 9: Aggregate confidence bounded by inputs**
    - For any non-empty list of scores in `[0.0, 1.0]`, `combine_confidence()` result is in `[min(scores), max(scores)]`
    - Use `st.lists(st.floats(0.0, 1.0), min_size=1)` as generator
    - **Validates: Requirements 12.6, 12.7, 12.8**

- [ ] 5. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Implement Audit Interceptor
  - [ ] 6.1 Implement audit interceptor in `base_skill_framework/audit.py`
    - Create `AuditInterceptor` class with `__init__(user_context, integrity_mode, sink, seed)` constructor
    - Implement `log(tool_name, status, **kwargs) -> AuditRecord` method that creates and emits audit records as JSON
    - Support status values: STARTED, COMPLETED, FAILED, TIMEOUT, CEILING_VIOLATION, PARTIAL_FAILURE, CB_STATE_CHANGE, HOOK_ALLOW, HOOK_DENY
    - Implement `_compute_hash(record_json)` and `_chain_hash(record)` for hash-chain mode
    - In "none" mode: JSON-serialize and emit via `logging.info()`
    - In "hash_chain" mode: compute `prev_hash` from last record, compute `record_hash`, then emit
    - First record in hash_chain mode uses the seed value (default: SHA-256 of session start + user_id)
    - Implement static `verify_audit_chain(records: list[dict]) -> bool` method
    - _Requirements: 4.1, 4.2, 4.3, 4.5, 15.3, 16.1, 16.2, 16.3, 16.4, 21.7_

  - [ ]* 6.2 Write property test for audit record completeness
    - **Property 5: Audit record completeness**
    - For any tool invocation, audit records contain ISO-8601 timestamp, `user_id`, `auth_method`, `tool_name`, and `status`
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.5, 4.6, 7.2, 15.3**

  - [ ]* 6.3 Write property test for audit hash chain tamper detection
    - **Property 13: Audit hash chain tamper detection**
    - Valid chains pass `verify_audit_chain()`; any single mutation (alter, remove, reorder) causes it to return `False`
    - Use random audit record sequences + random mutations
    - **Validates: Requirements 16.2, 16.4, 16.5**

- [ ] 7. Implement Circuit Breaker
  - [ ] 7.1 Implement circuit breaker in `base_skill_framework/circuit_breaker.py`
    - Create `CircuitState` enum with CLOSED, OPEN, HALF_OPEN values
    - Create `ToolCircuitState` dataclass with `state`, `consecutive_failures`, `last_failure_time`
    - Create `CircuitBreaker` class with `__init__(failure_threshold, recovery_timeout_seconds)` constructor
    - Implement `check(tool_name)` — raises `CircuitBreakerOpenError` if open and recovery timeout not elapsed
    - Implement `record_success(tool_name)` — resets to CLOSED
    - Implement `record_failure(tool_name)` — increments failures, transitions to OPEN at threshold
    - Implement `get_state(tool_name)` — returns current `CircuitState`
    - Handle HALF_OPEN state: allow one probe call, close on success, reopen on failure
    - _Requirements: 19.3, 19.4, 19.5, 19.6_

  - [ ]* 7.2 Write property test for circuit breaker state machine
    - **Property 16: Circuit breaker state machine**
    - After `failure_threshold` consecutive failures, circuit transitions to OPEN
    - After `recovery_timeout_seconds`, next call is allowed as probe
    - Probe success closes circuit; probe failure reopens it
    - Use random sequences of success/failure calls
    - **Validates: Requirements 19.2, 19.4, 19.5, 19.6**

- [ ] 8. Implement Audited Tool Decorator
  - [ ] 8.1 Implement `@audited_tool` decorator in `base_skill_framework/decorators.py`
    - Create `audited_tool(tool_name=None, timeout_seconds=30)` decorator function
    - Wrap decorated method with: circuit breaker check → audit STARTED → timeout-enforced execution → audit COMPLETED/FAILED/TIMEOUT → HWM registration → circuit breaker state update
    - Use `concurrent.futures.ThreadPoolExecutor` with `future.result(timeout=timeout_seconds)` for timeout enforcement
    - Default `tool_name` to the decorated method's `__name__` if not provided
    - Preserve original return value and exception behavior (audit must not alter functional outcome)
    - Ensure composability with other standard Python decorators via `functools.wraps`
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5, 4.4, 4.6, 19.1, 19.2_

- [ ] 9. Implement SecurityPolicyEngine
  - [ ] 9.1 Implement `SecurityPolicyEngine` in `base_skill_framework/security_policy.py`
    - Create `SecurityPolicyEngine` class with all methods as `@staticmethod` (no `__init__`, no instance state)
    - Implement `check_classification_ceiling(classification, ceiling, tool_name, source)` — raises `ClassificationCeilingExceededError` if classification > ceiling
    - Implement `validate_fragment_metadata(fragment)` — validates classification field (raises `ClassificationMissingError` / `ClassificationInvalidError`), source field with non-empty title and system (raises `SourceMissingError`), confidence in [0.0, 1.0] (raises `ValueError`)
    - Implement `validate_portion_marks(text, hwm)` — validates every paragraph's portion mark does not exceed HWM (raises `ClassificationExceedsHWMError`)
    - Implement `validate_citations(text, fragments)` — validates every paragraph referencing retrieved data contains at least one inline citation
    - Implement `cross_validate_marks_and_citations(text, fragments)` — cross-validates portion mark ≥ max classification of cited sources (raises `ClassificationMarkCitationMismatchError`)
    - All context passed as parameters — stateless and thread-safe
    - Raises the same `BaseSkillError` subclasses regardless of call path
    - _Requirements: 23.1, 23.2, 23.4, 23.5, 23.6, 2.2, 2.4, 5.6, 11.2, 13.2, 18.1, 18.2_

  - [ ]* 9.2 Write property test for SecurityPolicyEngine consistency
    - **Property 18: SecurityPolicyEngine consistency**
    - For any input (classification level and ceiling pair, data fragment, response text with portion marks, or response text with citations), the `SecurityPolicyEngine` shall raise the same exception types and produce the same validation outcomes regardless of call path
    - For any two calls to the same `SecurityPolicyEngine` static method with identical parameters, the results (return value or raised exception) shall be identical, confirming statelessness
    - Use random `DataFragment` + random `ClassificationLevel` pairs, called via both `BaseSkill` and `HookAdapter` paths
    - **Validates: Requirements 23.2, 23.3, 23.5, 23.6**

- [ ] 10. Implement Attribution and Citation Enforcers
  - [ ] 10.1 Implement attribution enforcer in `base_skill_framework/attribution.py`
    - Create `AttributionEnforcer` class with static methods: `enforce_portion_marks(text, hwm)`, `validate_portion_marks(text, hwm)`
    - Split text on `\n\n` to identify paragraphs
    - Use regex `^\([A-Z/]+\)` to detect existing portion marks
    - Prepend HWM portion mark to paragraphs missing a mark
    - Raise `ClassificationExceedsHWMError` if any mark exceeds HWM
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6_

  - [ ]* 10.2 Write property tests for attribution enforcement
    - **Property 6: Portion mark enforcement idempotence**
    - After enforcement, every paragraph starts with a valid mark; applying enforcement again produces identical output
    - **Property 7: Portion mark cannot exceed HWM**
    - Paragraphs with marks above HWM raise `ClassificationExceedsHWMError`
    - **Validates: Requirements 5.2, 5.3, 5.4, 5.5, 5.6**

  - [ ] 10.3 Implement citation enforcer in `base_skill_framework/citation.py`
    - Create `CitationEnforcer` class with static methods: `format_citation(source)`, `enforce_citations(text, fragments)`, `build_sources_appendix(fragments)`, `cross_validate(text, fragments)`
    - `format_citation()` produces `[Source: <title>, <system>]`
    - `enforce_citations()` scans paragraphs for inline citations; appends missing ones
    - `build_sources_appendix()` generates trailing sources list with classification levels
    - `cross_validate()` checks each paragraph's portion mark ≥ max classification of cited sources; raises `ClassificationMarkCitationMismatchError` on violation
    - _Requirements: 11.3, 11.4, 11.5, 11.6, 18.1, 18.2, 18.3_

  - [ ]* 10.4 Write property tests for citation enforcement
    - **Property 8: Citation round-trip**
    - For any `SourceMetadata` with non-empty `title` and `system`, `format_citation()` → extract title/system produces original values
    - **Property 15: Cross-validation of portion marks and citations**
    - Each paragraph's portion mark ≥ max classification of cited sources; violations raise `ClassificationMarkCitationMismatchError`
    - **Validates: Requirements 11.3, 11.7, 18.1, 18.2, 18.4**

- [ ] 11. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 12. Implement BaseSkill ABC
  - [ ] 12.1 Implement the `BaseSkill` abstract base class in `base_skill_framework/base_skill.py`
    - Define `BaseSkill(ABC)` with constructor accepting: `skill_name`, `default_classification`, `user_context`, `max_allowed_classification`, `failure_mode`, `audit_integrity_mode`, `confidence_threshold`, `default_confidence`, `confidence_penalty`, `circuit_breaker_config`
    - Validate `user_context` at construction — raise `AuthenticationRequiredError` if missing or incomplete
    - Set `max_allowed_classification` with precedence: explicit param > `user_context.clearance_level` > `default_classification` (log warning for fallback)
    - Initialize `_hwm_tracker`, `_audit_interceptor`, `_circuit_breaker`, `_retrieval_errors`
    - Implement `__init_subclass__` hook to seal `execute()` — raise `TypeError` if subclass defines it
    - Define abstract methods: `retrieve_data(query)`, `process_data(data)`, `format_response(processed_text)`
    - _Requirements: 1.1, 1.2, 1.3, 6.1, 6.2, 6.3, 6.4, 6.5, 13.1, 13.4, 14.1, 15.1, 15.2, 15.4_

  - [ ] 12.2 Implement the sealed `execute()` method and internal pipeline — delegates to SecurityPolicyEngine
    - Implement `execute(query)` orchestrating: audit STARTED → `retrieve_data()` → validate fragments → check ceiling → `process_data()` → `format_response()` → security envelope → audit COMPLETED
    - Implement `_validate_fragments()` — delegates to `SecurityPolicyEngine.validate_fragment_metadata()` for each fragment
    - Implement `_check_classification_ceiling()` — delegates to `SecurityPolicyEngine.check_classification_ceiling()`
    - Implement `_apply_security_envelope()` — delegates portion mark validation to `SecurityPolicyEngine.validate_portion_marks()`, citation validation to `SecurityPolicyEngine.validate_citations()`, cross-validation to `SecurityPolicyEngine.cross_validate_marks_and_citations()`
    - Implement `_enforce_portion_marks()` via `AttributionEnforcer`, `_enforce_citations()` via `CitationEnforcer`
    - Implement `_handle_empty_result()` — produce default response with banner, portion mark, empty appendix, confidence 0.0
    - In `"graceful"` mode: catch per-fragment errors, log PARTIAL_FAILURE, continue; raise `TotalRetrievalFailureError` if all fail
    - Include "Degraded Sources" section in output when partial failures occur
    - Reduce aggregate confidence by penalty factor per failed source
    - Log warning when contract methods are called outside `execute()` context
    - _Requirements: 1.4, 2.1, 2.2, 2.3, 2.4, 3.3, 3.4, 3.5, 5.1, 5.2, 11.4, 11.5, 11.6, 12.2, 12.3, 12.4, 12.5, 13.2, 13.3, 13.5, 14.2, 14.3, 17.1, 17.2, 17.3, 17.4, 17.5, 17.6, 20.1, 20.2, 20.3, 20.4, 20.5, 23.2_

  - [ ]* 12.3 Write property tests for BaseSkill contract enforcement
    - **Property 11: Execute method is sealed**
    - Subclass defining `execute()` raises `TypeError` at class definition time; subclass missing contract methods raises `TypeError` at instantiation
    - **Property 4: Data fragment validation rejects invalid fragments**
    - Fragments missing classification, with invalid classification, missing source, or with confidence outside [0.0, 1.0] are rejected with appropriate errors
    - **Validates: Requirements 1.2, 2.1, 2.3, 2.4, 11.1, 11.2, 12.1, 12.3, 14.1**

  - [ ]* 12.4 Write property test for classification ceiling enforcement
    - **Property 10: Classification ceiling enforcement**
    - Fragments above ceiling are rejected with `ClassificationCeilingExceededError`, not registered with HWM, and logged as CEILING_VIOLATION
    - **Validates: Requirements 13.2, 13.3, 13.5**

  - [ ]* 12.5 Write property test for graceful degradation
    - **Property 14: Graceful degradation preserves successful results**
    - In graceful mode with at least one success: execution continues, failed sources excluded from HWM, "Degraded Sources" section included, confidence reduced by penalty
    - **Validates: Requirements 17.2, 17.3, 17.4, 17.6**

  - [ ]* 12.6 Write property test for empty result handling
    - **Property 17: Empty result produces well-formed default response**
    - Empty/whitespace input produces response with banner, portion mark, empty appendix, confidence 0.0, and disclaimer
    - **Validates: Requirements 20.1, 20.3, 20.4, 20.5**

- [ ] 13. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 14. Implement HookAdapter Interface
  - [ ] 14.1 Implement `HookAdapter` abstract base class in `base_skill_framework/hooks.py`
    - Create `HookAdapter(ABC)` with `__init__(policy_engine: SecurityPolicyEngine, audit_interceptor: AuditInterceptor)`
    - Define abstract methods: `pre_tool_call(tool_name, arguments, user_context) -> HookDecision`, `post_tool_call(tool_name, result, user_context) -> HookDecision`, `on_execution_complete(response, user_context) -> HookDecision`
    - `pre_tool_call` delegates ceiling check to `SecurityPolicyEngine.check_classification_ceiling()`
    - `post_tool_call` delegates metadata validation to `SecurityPolicyEngine.validate_fragment_metadata()`
    - `on_execution_complete` delegates portion mark validation to `SecurityPolicyEngine.validate_portion_marks()` and citation validation to `SecurityPolicyEngine.validate_citations()`
    - Implement concrete `_log_decision(tool_name, decision, user_context)` helper that logs `HOOK_ALLOW` or `HOOK_DENY` via `AuditInterceptor`
    - `hooks.py` must NOT import any orchestrator-specific library (no LangGraph, no CrewAI, etc.)
    - _Requirements: 21.1, 21.3, 21.4, 21.5, 21.6, 21.7, 23.3_

  - [ ]* 14.2 Write property test for HookAdapter decision logging
    - **Property 19: HookAdapter decision logging**
    - For any hook invocation (`pre_tool_call`, `post_tool_call`, or `on_execution_complete`) on any `HookAdapter` implementation, the adapter shall produce exactly one audit record with status `HOOK_ALLOW` or `HOOK_DENY`, including the tool name, the decision reason, and the `user_context.user_id`
    - Use random tool names + random `UserContext` + random arguments/results
    - **Validates: Requirements 21.7**

  - [ ]* 14.3 Write unit tests for HookAdapter interface
    - Test that `HookAdapter` cannot be instantiated directly (raises `TypeError`)
    - Test that subclasses missing abstract methods raise `TypeError` at instantiation
    - Verify `HookDecision` dataclass construction with required and optional fields
    - Verify `hooks.py` does not import orchestrator-specific modules
    - _Requirements: 21.1, 21.2, 21.6_

- [ ] 15. Implement LangGraphHookAdapter
  - [ ] 15.1 Implement `LangGraphHookAdapter` in `base_skill_framework/adapters/langgraph.py`
    - Create `LangGraphHookAdapter(HookAdapter)` extending the abstract `HookAdapter`
    - Implement lazy import of `langchain_core` via `_lazy_import_langchain_core()` — only imported at `__init__()` time, not at module level
    - Implement `pre_tool_call()` — extracts expected classification from arguments, delegates ceiling check to `SecurityPolicyEngine`, logs decision, returns `HookDecision`
    - Implement `post_tool_call()` — delegates metadata validation to `SecurityPolicyEngine`, logs decision, returns `HookDecision`
    - Implement `on_execution_complete()` — delegates portion mark + citation validation to `SecurityPolicyEngine`, logs decision, returns `HookDecision`
    - Implement `as_langraph_callbacks(user_context)` — returns dict with keys `on_tool_start`, `on_tool_end`, `on_chain_end` mapping to callback handlers that inject `user_context` via LangGraph `RunnableConfig.metadata`
    - Raise helpful `ImportError` if `langchain-core` is not installed
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5, 22.6, 22.7_

  - [ ]* 15.2 Write integration tests for LangGraphHookAdapter
    - Test that `LangGraphHookAdapter` extends `HookAdapter` and can be instantiated (with mock langchain_core)
    - Test `pre_tool_call` → `on_tool_start` callback translation
    - Test `post_tool_call` → `on_tool_end` callback translation
    - Test `on_execution_complete` → `on_chain_end` callback translation
    - Test `user_context` propagation via LangGraph `RunnableConfig.metadata`
    - Test that core `base_skill_framework` imports succeed without LangGraph installed
    - Test that `LangGraphHookAdapter` import from `base_skill_framework.adapters.langgraph` works
    - Test that instantiation without `langchain-core` raises helpful `ImportError`
    - _Requirements: 22.1, 22.2, 22.3, 22.4, 22.5, 22.6, 22.7_

  - [ ]* 15.3 Write integration tests for SecurityPolicyEngine delegation
    - Test that `BaseSkill.execute()` delegates validation to `SecurityPolicyEngine` methods (mock `SecurityPolicyEngine`, verify calls for ceiling checks, fragment validation, portion marks, citations, cross-validation)
    - Test that `HookAdapter` methods delegate to the same `SecurityPolicyEngine` methods
    - Verify that a policy change in `SecurityPolicyEngine` applies to both `BaseSkill` and `HookAdapter` paths without duplicate modifications
    - _Requirements: 23.2, 23.3, 23.4_

- [ ] 16. Checkpoint
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 17. Implement ExampleConfluenceSkill
  - [ ] 17.1 Implement `ExampleConfluenceSkill` in `base_skill_framework/example_confluence_skill.py`
    - Extend `BaseSkill` and implement all three contract methods
    - Apply `@audited_tool(tool_name="confluence_search")` to `retrieve_data()`
    - `retrieve_data()` invokes an imaginary `confluence_search` MCP tool and returns `DataFragment` objects with classification, source, and confidence
    - `process_data()` concatenates content fragments preserving portion marks
    - `format_response()` returns text with portion marks on every paragraph
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

  - [ ]* 17.2 Write integration tests for ExampleConfluenceSkill
    - Test full `execute()` pipeline with mock MCP tool returning various classification levels
    - Verify complete output structure: banner + portion marks + citations + sources appendix + confidence
    - Test graceful degradation with partial MCP failures
    - Test circuit breaker integration with consecutive failures
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [ ] 18. Create Steering File
  - [ ] 18.1 Create `.kiro/steering/base-skill-security.md`
    - Instruct agents to extend `BaseSkill` for every new Skill
    - Instruct agents to apply `@audited_tool` to every MCP method
    - Instruct agents to include `classification` field in every data retrieval response
    - Instruct agents to apply portion marks to every paragraph in generated output
    - Include a security compliance checklist that agents must verify before considering a Skill complete
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

- [ ] 19. Wire up package exports and final integration
  - [ ] 19.1 Update `base_skill_framework/__init__.py` with all public exports
    - Export all public classes: `BaseSkill`, `ClassificationLevel`, `DataFragment`, `SourceMetadata`, `UserContext`, `AuditRecord`, `HookDecision`, `AuditInterceptor`, `HighWaterMarkTracker`, `AttributionEnforcer`, `CitationEnforcer`, `ConfidenceScorer`, `CircuitBreaker`, `SecurityPolicyEngine`, `HookAdapter`, `ExampleConfluenceSkill`
    - Export the `audited_tool` decorator
    - Export all exception classes
    - Export utility functions: `parse_classification`, `compare_classifications`
    - Note: `LangGraphHookAdapter` is NOT exported from the top-level package — it is imported from `base_skill_framework.adapters.langgraph`
    - _Requirements: 1.1, 6.5, 22.6, 22.7_

  - [ ]* 19.2 Write unit tests for decorator composability and direct-call warnings
    - Test `@audited_tool` composability with `@staticmethod` and custom decorators
    - Test that calling `retrieve_data()` outside `execute()` logs a warning
    - Test configuration precedence: explicit `max_allowed_classification` > `clearance_level` > `default_classification`
    - _Requirements: 7.5, 14.3, 13.4, 15.4_

- [ ] 20. Final checkpoint
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation after each major phase
- Property tests validate the 19 universal correctness properties defined in the design
- Unit and integration tests cover specific examples, configuration, and end-to-end flows
- All code is Python; uses `hypothesis` for property-based testing and `pytest` as the test runner
- SecurityPolicyEngine (task 9) is implemented BEFORE BaseSkill ABC (task 12) because BaseSkill delegates validation to it
- HookAdapter (task 14) is implemented AFTER SecurityPolicyEngine (task 9) and AuditInterceptor (task 6) since it depends on both
- LangGraphHookAdapter (task 15) is implemented AFTER HookAdapter (task 14) as a concrete subclass
- LangGraphHookAdapter lives in `base_skill_framework/adapters/langgraph.py` and is NOT exported from the top-level package to avoid coupling the core framework to LangGraph

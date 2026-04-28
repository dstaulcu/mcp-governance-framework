# Requirements Document

## Introduction

This document defines the requirements for a BaseSkill framework designed for high-security (Intel/Defense) environments. The framework provides a modular Skill library where security governance, classification marking, audit logging, and attribution enforcement are inherited by default through a base class pattern. Developers building new Skills (e.g., Jira Skill, Confluence Skill) extend the BaseSkill rather than forking code, ensuring consistent security posture across all Skills.

## Glossary

- **BaseSkill**: An abstract Python class that encapsulates mandatory security behaviors (audit logging, classification marking, high-water mark calculation, attribution enforcement) and defines the contract that all specialized Skills must implement.
- **Skill**: A concrete implementation that extends BaseSkill to integrate with a specific external system (e.g., Confluence, Jira) via MCP tool calls.
- **MCP**: Model Context Protocol — the interface through which Skills retrieve data from external systems.
- **Portion_Mark**: A parenthetical classification label applied to individual paragraphs or data fragments, following the format (U), (C), (S), (TS), or (TS//SCI).
- **Classification_Level**: One of the discrete security levels in ascending order: UNCLASSIFIED (U), CONFIDENTIAL (C), SECRET (S), TOP_SECRET (TS), TOP_SECRET_SCI (TS//SCI).
- **High_Water_Mark**: The highest Classification_Level found among all data sources consulted during a single Skill execution, used to determine the overall response banner.
- **Response_Banner**: A top-level classification label applied to the entire response output, derived from the High_Water_Mark of all consulted sources.
- **Audit_Record**: A structured log entry containing Timestamp, UserID, ToolName, DataClassification, and execution metadata for a single tool invocation.
- **Audit_Interceptor**: A middleware component that automatically generates an Audit_Record before and after every tool execution without requiring explicit developer invocation.
- **Skill_Contract**: A Python interface (abstract methods) that every concrete Skill must implement to be considered a valid Skill.
- **Steering_File**: A .kiro-steering markdown file that instructs future AI agents to follow the security patterns defined by this framework.
- **Decorator_Pattern**: A structural design pattern used to wrap tool execution methods with automatic audit and classification logic.
- **Source_Metadata**: A structured record containing the origin of a data fragment, including fields such as document title, URL, or system of record identifier.
- **Inline_Citation**: A parenthetical reference embedded within a paragraph that attributes a claim or data point to its originating source, following the format `[Source: <title>, <system>]`.
- **Sources_Appendix**: A section appended to the final response output that lists all consulted sources along with their Classification_Levels.
- **SourceMissingError**: An exception raised when a data retrieval response lacks the required `source` field.
- **Confidence_Score**: A floating-point value between 0.0 and 1.0 (inclusive) representing the assessed reliability of a data fragment, where 0.0 indicates no confidence and 1.0 indicates full confidence.
- **Aggregate_Confidence**: A single Confidence_Score computed from all individual fragment Confidence_Scores for a given Skill execution, representing the overall reliability of the response.
- **Confidence_Disclaimer**: A textual warning prepended to the response output when the Aggregate_Confidence falls below a configurable threshold, identifying the overall score and lowest-confidence sources.
- **max_allowed_classification**: A Classification_Level parameter provided at BaseSkill construction representing the maximum classification ceiling for the current user session; data fragments exceeding this level are rejected.
- **ClassificationCeilingExceededError**: An exception raised when a data fragment's Classification_Level exceeds the configured max_allowed_classification for the session.
- **user_context**: An immutable structured parameter provided at BaseSkill construction containing authenticated user identity information: `user_id` (str), `auth_method` (str), and `clearance_level` (Classification_Level).
- **AuthenticationRequiredError**: An exception raised at BaseSkill construction time when `user_context` is not provided or is missing required fields (`user_id`, `auth_method`).
- **audit_integrity_mode**: A configurable setting controlling the integrity guarantees of Audit_Records, with options: "none" (default Python logging), "hash_chain" (SHA-256 chained hashes), and "signed" (cryptographically signed records).
- **hash_chain**: An audit integrity mechanism where each Audit_Record includes a SHA-256 hash of the previous record's JSON representation, enabling tamper detection across the audit log sequence.
- **failure_mode**: A configurable parameter controlling how the BaseSkill handles partial data retrieval failures, with options: "fail_fast" (default — any error aborts execution) and "graceful" (partial failures are captured and execution continues).
- **retrieval_errors**: An internal collection maintained by the BaseSkill during "graceful" failure_mode execution, capturing error details for each data retrieval that failed.
- **TotalRetrievalFailureError**: An exception raised when all data retrieval attempts fail during a Skill execution, even when failure_mode is set to "graceful".
- **ClassificationMarkCitationMismatchError**: An exception raised when a paragraph's Portion_Mark is lower than the Classification_Level of any source cited within that paragraph, indicating a classification marking violation.
- **ToolTimeoutError**: An exception raised when a method decorated with `@audited_tool` exceeds its configured `timeout_seconds` limit.
- **CircuitBreakerOpenError**: An exception raised when a call is made to a tool whose circuit breaker is in the open state, indicating the tool has exceeded its consecutive failure threshold.
- **circuit_breaker**: A resilience mechanism that tracks consecutive failures for each tool and temporarily blocks further calls after a configurable failure threshold is reached, allowing recovery after a timeout period.
- **HookAdapter**: An abstract interface defining orchestrator-agnostic hook methods (`pre_tool_call`, `post_tool_call`, `on_execution_complete`) that translate BaseSkill security policies into orchestrator-level pre/post tool callbacks, enabling dual-layer enforcement without coupling the framework to any specific orchestrator.
- **HookDecision**: A dataclass returned by HookAdapter methods containing: `allow` (bool), `reason` (str), `modified_arguments` (optional dict for pre-hook argument rewriting), and `modified_result` (optional for post-hook result filtering), representing the outcome of a hook evaluation.
- **SecurityPolicyEngine**: A stateless class that encapsulates all reusable security checks (classification ceiling, fragment metadata validation, portion mark compliance, citation completeness, cross-validation) in a single location, callable by both the BaseSkill execution pipeline and any HookAdapter implementation.
- **LangGraphHookAdapter**: A concrete implementation of HookAdapter that translates the abstract hook interface into LangGraph-specific callbacks (`on_tool_start`, `on_tool_end`, `on_chain_end`), provided as an optional subpackage to avoid coupling the core framework to LangGraph.

## Requirements

### Requirement 1: Abstract BaseSkill Class

**User Story:** As a Skill developer, I want an abstract BaseSkill class that enforces security behaviors by default, so that I cannot accidentally build a Skill that bypasses classification, audit, or attribution rules.

#### Acceptance Criteria

1. THE BaseSkill SHALL define abstract methods that constitute the Skill_Contract: `retrieve_data()`, `process_data()`, and `format_response()`.
2. THE BaseSkill SHALL raise a TypeError when a developer attempts to instantiate it directly without implementing all Skill_Contract methods.
3. THE BaseSkill SHALL accept a `skill_name` string and a `default_classification` Classification_Level as constructor parameters.
4. THE BaseSkill SHALL provide a concrete `execute()` method that orchestrates the call sequence: retrieve_data → process_data → format_response, with Audit_Interceptor and High_Water_Mark logic applied automatically.

### Requirement 2: Mandatory Metadata on Data Retrieval

**User Story:** As a security officer, I want every piece of data retrieved via MCP to carry a classification Portion_Mark, so that downstream processing always knows the sensitivity of each data fragment.

#### Acceptance Criteria

1. WHEN a Skill retrieves data via MCP, THE BaseSkill SHALL require the returned data structure to include a `classification` field containing a valid Classification_Level.
2. IF a data retrieval response lacks a `classification` field, THEN THE BaseSkill SHALL reject the response and raise a ClassificationMissingError with a message identifying the source and tool name.
3. THE BaseSkill SHALL validate that the `classification` field value is one of the defined Classification_Level values: U, C, S, TS, TS//SCI.
4. IF a data retrieval response contains an unrecognized `classification` value, THEN THE BaseSkill SHALL reject the response and raise a ClassificationInvalidError with the invalid value and the list of accepted values.

### Requirement 3: High-Water Mark Calculation

**User Story:** As a security officer, I want the framework to automatically calculate the highest classification level across all data sources consulted during a Skill execution, so that the final response is labeled at the correct overall classification.

#### Acceptance Criteria

1. THE BaseSkill SHALL maintain an ordered ranking of Classification_Levels: U < C < S < TS < TS//SCI.
2. WHEN a Skill retrieves data from one or more sources, THE BaseSkill SHALL track the Classification_Level of each source in an internal collection.
3. WHEN the Skill execution completes, THE BaseSkill SHALL compute the High_Water_Mark as the maximum Classification_Level from all tracked sources.
4. THE BaseSkill SHALL prepend the computed High_Water_Mark as a Response_Banner to the final output in the format: `// OVERALL CLASSIFICATION: <HIGH_WATER_MARK> //`.
5. IF no data sources are consulted during execution, THEN THE BaseSkill SHALL default the High_Water_Mark to the `default_classification` provided at construction.
6. FOR ALL combinations of Classification_Levels from multiple sources, computing the High_Water_Mark and then adding a source at or below that level SHALL produce the same High_Water_Mark (idempotence of maximum).

### Requirement 4: Audit Interceptor

**User Story:** As a compliance auditor, I want every tool execution to be automatically logged in a standard format, so that I have a complete and tamper-evident record of all data access.

#### Acceptance Criteria

1. WHEN a tool execution begins, THE Audit_Interceptor SHALL generate an Audit_Record containing: ISO-8601 Timestamp, UserID, ToolName, and a status of "STARTED".
2. WHEN a tool execution completes successfully, THE Audit_Interceptor SHALL generate an Audit_Record containing: ISO-8601 Timestamp, UserID, ToolName, DataClassification of the result, and a status of "COMPLETED".
3. IF a tool execution fails with an exception, THEN THE Audit_Interceptor SHALL generate an Audit_Record containing: ISO-8601 Timestamp, UserID, ToolName, error description, and a status of "FAILED".
4. THE Audit_Interceptor SHALL apply automatically to every tool execution via the Decorator_Pattern without requiring the Skill developer to add explicit logging calls.
5. THE Audit_Interceptor SHALL write Audit_Records to a configurable output sink (default: standard Python logging at INFO level) as JSON-formatted strings.
6. THE Audit_Interceptor SHALL preserve the original return value and exception behavior of the wrapped tool execution — logging SHALL NOT alter the functional outcome.

### Requirement 5: Attribution Enforcement

**User Story:** As a security officer, I want the framework to force the LLM-generated response to include Portion_Marks on every paragraph, so that readers can identify the classification of each piece of information at a glance.

#### Acceptance Criteria

1. WHEN the `format_response()` method produces output text, THE BaseSkill SHALL scan each paragraph for a leading Portion_Mark.
2. IF a paragraph in the output text lacks a leading Portion_Mark, THEN THE BaseSkill SHALL prepend the High_Water_Mark as the Portion_Mark for that paragraph.
3. THE BaseSkill SHALL recognize a paragraph as any block of text separated by one or more blank lines.
4. THE BaseSkill SHALL recognize a valid leading Portion_Mark as a string matching the pattern `\([A-Z/]+\)` at the start of a paragraph, corresponding to a defined Classification_Level.
5. IF a paragraph already contains a valid leading Portion_Mark, THEN THE BaseSkill SHALL retain the existing Portion_Mark unchanged.
6. IF a paragraph contains a Portion_Mark that exceeds the computed High_Water_Mark, THEN THE BaseSkill SHALL raise a ClassificationExceedsHWMError identifying the paragraph and the conflicting levels.

### Requirement 6: Skill Contract Interface

**User Story:** As a platform architect, I want a clearly defined interface that all specialized Skills must implement, so that the framework can guarantee consistent behavior and security compliance across all Skills.

#### Acceptance Criteria

1. THE Skill_Contract SHALL define `retrieve_data(query: str) -> list[dict]` as an abstract method that returns a list of data fragments, each containing at minimum a `content` field and a `classification` field.
2. THE Skill_Contract SHALL define `process_data(data: list[dict]) -> str` as an abstract method that accepts retrieved data fragments and returns processed text.
3. THE Skill_Contract SHALL define `format_response(processed_text: str) -> str` as an abstract method that accepts processed text and returns the final user-facing output.
4. IF a concrete Skill class does not implement all three Skill_Contract methods, THEN THE Python runtime SHALL raise a TypeError at class instantiation time.
5. THE Skill_Contract SHALL be implemented as a Python Abstract Base Class (ABC) with `@abstractmethod` decorators.

### Requirement 7: Decorator-Based Middleware for Automatic Security Enforcement

**User Story:** As a Skill developer, I want audit logging and High_Water_Mark tracking to run automatically around my tool calls, so that I can focus on business logic without manually wiring security concerns.

#### Acceptance Criteria

1. THE BaseSkill SHALL provide a `@audited_tool` decorator that Skill developers apply to any method that performs an MCP tool call.
2. WHEN a method decorated with `@audited_tool` is invoked, THE Decorator_Pattern SHALL execute the Audit_Interceptor logic (Requirement 4) before and after the method body.
3. WHEN a method decorated with `@audited_tool` returns data containing a `classification` field, THE Decorator_Pattern SHALL automatically register that Classification_Level with the High_Water_Mark tracker (Requirement 3).
4. THE `@audited_tool` decorator SHALL accept an optional `tool_name` parameter; WHEN not provided, THE decorator SHALL default to the decorated method's function name.
5. THE `@audited_tool` decorator SHALL be composable with other standard Python decorators without altering their behavior.

### Requirement 8: Example Confluence Skill Implementation

**User Story:** As a Skill developer, I want a reference implementation of a Confluence Skill that extends BaseSkill, so that I have a concrete example of how to build a compliant Skill.

#### Acceptance Criteria

1. THE ExampleConfluenceSkill SHALL extend BaseSkill and implement all three Skill_Contract methods.
2. THE ExampleConfluenceSkill SHALL use the `@audited_tool` decorator on its `retrieve_data()` method.
3. WHEN `retrieve_data()` is called with a search query, THE ExampleConfluenceSkill SHALL invoke an imaginary Confluence MCP tool named `confluence_search` and return results with Portion_Marks.
4. WHEN `process_data()` is called, THE ExampleConfluenceSkill SHALL concatenate retrieved content fragments into a summary, preserving each fragment's Portion_Mark.
5. WHEN `format_response()` is called, THE ExampleConfluenceSkill SHALL return the processed text with Portion_Marks on every paragraph.

### Requirement 9: Steering File for Future Agent Compliance

**User Story:** As a platform architect, I want a Steering File that instructs future AI agents to follow the BaseSkill security patterns, so that new Skills generated by agents are compliant by default.

#### Acceptance Criteria

1. THE Steering_File SHALL be located at `.kiro/steering/base-skill-security.md` in the project repository.
2. THE Steering_File SHALL instruct agents to extend BaseSkill for every new Skill implementation.
3. THE Steering_File SHALL instruct agents to apply the `@audited_tool` decorator to every method that performs an MCP tool call.
4. THE Steering_File SHALL instruct agents to include a `classification` field in every data retrieval response.
5. THE Steering_File SHALL instruct agents to apply Portion_Marks to every paragraph in generated output.
6. THE Steering_File SHALL include a checklist of security compliance items that agents must verify before considering a Skill implementation complete.

### Requirement 10: Classification Data Model and Validation

**User Story:** As a Skill developer, I want a well-defined data model for classification levels and portion marks, so that I can work with classification metadata in a type-safe and consistent manner.

#### Acceptance Criteria

1. THE BaseSkill SHALL define Classification_Level as a Python Enum with values: U, C, S, TS, TS_SCI, each mapped to a display string and a numeric rank.
2. THE BaseSkill SHALL provide a `parse_classification(value: str) -> Classification_Level` utility that accepts common string representations (e.g., "U", "UNCLASSIFIED", "(U)", "TS//SCI") and returns the corresponding Enum member.
3. IF `parse_classification()` receives an unrecognized string, THEN THE BaseSkill SHALL raise a ClassificationInvalidError.
4. FOR ALL valid Classification_Level enum members, formatting a Classification_Level to its display string and then parsing it back SHALL produce the original Classification_Level (round-trip property).
5. THE BaseSkill SHALL provide a `compare_classifications(a: Classification_Level, b: Classification_Level) -> Classification_Level` utility that returns the higher of the two levels.


### Requirement 11: Source Citation Enforcement

**User Story:** As an intelligence analyst, I want every data fragment to carry source attribution metadata and every response to include inline citations, so that I can trace every claim back to its origin in compliance with Intel/Defense sourcing standards.

#### Acceptance Criteria

1. WHEN a Skill retrieves data via MCP, THE BaseSkill SHALL require the returned data structure to include a `source` field containing a Source_Metadata record (e.g., document title, URL, or system of record identifier) alongside the existing `classification` field.
2. IF a data retrieval response lacks a `source` field, THEN THE BaseSkill SHALL reject the response and raise a SourceMissingError with a message identifying the tool name and query.
3. THE BaseSkill SHALL provide a `format_citation(source: Source_Metadata) -> str` utility that generates an Inline_Citation string in the format `[Source: <title>, <system>]` from the provided Source_Metadata.
4. WHEN the `format_response()` method produces output text, THE BaseSkill SHALL verify that every paragraph referencing retrieved data contains at least one Inline_Citation.
5. IF a paragraph references retrieved data and lacks an Inline_Citation, THEN THE BaseSkill SHALL append an Inline_Citation referencing all sources that contributed to that paragraph's content.
6. WHEN the Skill execution completes, THE BaseSkill SHALL append a Sources_Appendix to the final response listing all consulted sources with their corresponding Classification_Levels.
7. FOR ALL Source_Metadata records, formatting a Source_Metadata into an Inline_Citation and then extracting the title and system fields from the citation string SHALL produce values matching the original Source_Metadata (round-trip property).

### Requirement 12: Confidence Scoring

**User Story:** As an intelligence analyst, I want every data fragment to carry a confidence score and the overall response to surface aggregate reliability, so that I can assess the trustworthiness of information in accordance with the intelligence source reliability and information confidence matrix.

#### Acceptance Criteria

1. WHEN a Skill retrieves data via MCP, THE BaseSkill SHALL require the returned data structure to include a `confidence` field containing a Confidence_Score (float between 0.0 and 1.0 inclusive).
2. IF a data retrieval response lacks a `confidence` field, THEN THE BaseSkill SHALL assign a default Confidence_Score from a configurable value (default: 0.5) and log a warning via the standard Python logging module identifying the tool name and query.
3. IF a data retrieval response contains a `confidence` value outside the range 0.0 to 1.0, THEN THE BaseSkill SHALL reject the response and raise a ValueError identifying the invalid value and the accepted range.
4. WHEN the Skill execution completes, THE BaseSkill SHALL compute an Aggregate_Confidence from all collected fragment Confidence_Scores using a configurable aggregation strategy (default: weighted average).
5. IF the Aggregate_Confidence falls below a configurable threshold (default: 0.7), THEN THE BaseSkill SHALL prepend a Confidence_Disclaimer to the response stating the Aggregate_Confidence value and identifying the sources with the lowest individual Confidence_Scores.
6. THE BaseSkill SHALL provide a `combine_confidence(scores: list[float], weights: list[float] | None) -> float` utility that computes an aggregate Confidence_Score from a list of individual scores with optional weights.
7. THE BaseSkill SHALL provide a `compare_confidence(a: float, b: float) -> float` utility that returns the higher of two Confidence_Scores.
8. FOR ALL non-empty lists of Confidence_Scores, the Aggregate_Confidence computed via weighted average SHALL be greater than or equal to the minimum individual score and less than or equal to the maximum individual score (bounded-by-inputs property).


### Requirement 13: Classification Ceiling / Access Control

**User Story:** As a security officer, I want to ensure that no Skill can retrieve or surface data above the current user's clearance level, so that the BaseSkill enforces a maximum allowed classification ceiling per session.

#### Acceptance Criteria

1. THE BaseSkill SHALL accept a `max_allowed_classification` parameter (Classification_Level) at construction, representing the user's clearance ceiling for the current session.
2. WHEN a data fragment's Classification_Level exceeds the configured `max_allowed_classification`, THE BaseSkill SHALL reject that fragment and raise a ClassificationCeilingExceededError with the fragment's classification, the ceiling, the tool name, and the source.
3. THE BaseSkill SHALL perform the classification ceiling check BEFORE the data fragment is passed to `process_data()` or registered with the High_Water_Mark tracker.
4. IF `max_allowed_classification` is not provided at construction, THEN THE BaseSkill SHALL default to the `default_classification` and log a warning via the standard Python logging module.
5. WHEN a classification ceiling violation occurs, THE Audit_Interceptor SHALL log the violation as a distinct Audit_Record with status "CEILING_VIOLATION", including the fragment's classification, the ceiling, the tool name, and the source.

### Requirement 14: Execute Method Tamper Prevention

**User Story:** As a platform architect, I want to guarantee that Skill developers cannot bypass the security envelope by overriding the `execute()` method or calling contract methods directly outside the orchestration flow, so that the security middleware is always active during Skill execution.

#### Acceptance Criteria

1. THE BaseSkill SHALL prevent subclasses from overriding the `execute()` method by using a `__init_subclass__` hook or equivalent mechanism to raise a TypeError if a subclass defines `execute()`.
2. THE BaseSkill SHALL ensure that the security post-processing (attribution enforcement, High_Water_Mark banner, citation verification, confidence disclaimer) runs as a final step that cannot be bypassed by the Skill developer's `format_response()` implementation.
3. IF a Skill developer attempts to call `retrieve_data()`, `process_data()`, or `format_response()` directly (outside of `execute()`), THEN THE BaseSkill SHALL log a warning via the standard Python logging module indicating that security middleware may not be active.
4. THE `execute()` method SHALL be the only supported entry point for Skill execution, and this requirement SHALL be documented in the Skill_Contract.

### Requirement 15: Authenticated UserID Provenance

**User Story:** As a security officer, I want audit records tied to a verified identity (e.g., CAC/PKI) rather than an arbitrary string, so that the BaseSkill provides a formal mechanism for authenticated user identity in defense environments.

#### Acceptance Criteria

1. THE BaseSkill SHALL accept a `user_context` parameter at construction containing at minimum: `user_id` (str), `auth_method` (str, e.g., "CAC", "PKI", "SAML"), and `clearance_level` (Classification_Level).
2. IF `user_context` is not provided or is missing required fields (`user_id`, `auth_method`), THEN THE BaseSkill SHALL raise an AuthenticationRequiredError at construction time.
3. THE Audit_Interceptor SHALL include the full `user_context.user_id` and `user_context.auth_method` in every Audit_Record.
4. IF `user_context.clearance_level` is provided and no explicit `max_allowed_classification` is also provided, THEN THE BaseSkill SHALL use `user_context.clearance_level` as the `max_allowed_classification`; an explicit `max_allowed_classification` parameter SHALL take precedence.
5. THE `user_context` SHALL be immutable after construction — attempts to modify the `user_context` attribute SHALL raise an AttributeError.

### Requirement 16: Audit Log Integrity

**User Story:** As a compliance auditor, I want tamper-evident audit logs with cryptographic integrity guarantees, so that audit records in defense environments cannot be silently altered or deleted.

#### Acceptance Criteria

1. THE BaseSkill SHALL support a configurable audit_integrity_mode with options: "none" (default Python logging), "hash_chain" (each record includes a SHA-256 hash of the previous record), and "signed" (each record is signed with a configurable key).
2. WHILE audit_integrity_mode is "hash_chain", THE Audit_Interceptor SHALL include a `prev_hash` field in each Audit_Record containing the SHA-256 hash of the previous Audit_Record's JSON representation, and a `record_hash` field containing the current record's own hash.
3. WHILE audit_integrity_mode is "hash_chain", THE Audit_Interceptor SHALL use a configurable seed value (default: SHA-256 of the session start timestamp concatenated with user_id) as the `prev_hash` for the first Audit_Record in a session.
4. THE BaseSkill SHALL provide a `verify_audit_chain(records: list[dict]) -> bool` utility that validates the hash chain integrity of a sequence of Audit_Records by recomputing each record's hash and verifying it against the next record's `prev_hash`.
5. FOR ALL valid audit chains, removing or modifying any single record SHALL cause `verify_audit_chain` to return False (tamper detection property).

### Requirement 17: Partial Failure and Graceful Degradation

**User Story:** As an intelligence analyst, I want partial failures during data retrieval to not cause total execution failure, so that I receive partial results with clear provenance rather than no results at all.

#### Acceptance Criteria

1. THE BaseSkill SHALL support a configurable `failure_mode` parameter with options: "fail_fast" (default — any error aborts execution) and "graceful" (partial failures are captured and execution continues with available data).
2. WHEN `failure_mode` is "graceful" and a data retrieval fails, THE BaseSkill SHALL capture the error in the `retrieval_errors` collection, log the failure via the Audit_Interceptor with status "PARTIAL_FAILURE", and continue processing remaining sources.
3. WHEN `failure_mode` is "graceful" and one or more retrievals failed, THE BaseSkill SHALL reduce the Aggregate_Confidence by a configurable penalty factor (default: 0.2 per failed source).
4. WHEN `failure_mode` is "graceful" and partial results are returned, THE BaseSkill SHALL include a "Degraded Sources" section in the final response listing which sources failed and the corresponding error descriptions.
5. IF all data retrievals fail (even when `failure_mode` is "graceful"), THEN THE BaseSkill SHALL raise a TotalRetrievalFailureError.
6. THE High_Water_Mark calculation SHALL only consider successfully retrieved sources — failed sources SHALL NOT contribute to the High_Water_Mark.

### Requirement 18: Cross-Validation of Portion Marks and Citation Classifications

**User Story:** As a security officer, I want the framework to cross-validate that portion marks are consistent with the classifications of cited sources, so that a paragraph marked at a lower level than its cited sources is detected as a classification violation.

#### Acceptance Criteria

1. WHEN the BaseSkill applies attribution enforcement (Requirement 5) and citation enforcement (Requirement 11), THE BaseSkill SHALL cross-validate that each paragraph's Portion_Mark is greater than or equal to the highest Classification_Level among that paragraph's cited sources.
2. IF a paragraph's Portion_Mark is lower than the Classification_Level of any of its cited sources, THEN THE BaseSkill SHALL raise a ClassificationMarkCitationMismatchError identifying the paragraph, its Portion_Mark, the conflicting source, and the source's Classification_Level.
3. THE cross-validation SHALL occur after both portion marks and citations have been applied, as a final integrity check before the response is returned.
4. FOR ALL paragraphs in a valid response, the paragraph's Portion_Mark SHALL be greater than or equal to the maximum Classification_Level of all sources cited in that paragraph (mark-citation consistency property).

### Requirement 19: Timeout and Circuit Breaker for MCP Tool Calls

**User Story:** As a Skill developer, I want protection against hanging or slow MCP tools with fail-closed timeout behavior and automatic circuit breaking, so that the system remains responsive and resilient in defense environments.

#### Acceptance Criteria

1. THE `@audited_tool` decorator SHALL accept an optional `timeout_seconds` parameter (default: 30) specifying the maximum allowed execution time for the decorated method.
2. IF a decorated method exceeds its `timeout_seconds`, THEN THE BaseSkill SHALL abort the call, raise a ToolTimeoutError, and log an Audit_Record with status "TIMEOUT".
3. THE BaseSkill SHALL support a configurable circuit_breaker with parameters: `failure_threshold` (default: 3) and `recovery_timeout_seconds` (default: 60).
4. WHEN a tool fails `failure_threshold` consecutive times, THE circuit_breaker SHALL open and immediately reject subsequent calls to that tool with a CircuitBreakerOpenError until `recovery_timeout_seconds` has elapsed.
5. WHEN the circuit_breaker is open and `recovery_timeout_seconds` elapses, THE circuit_breaker SHALL allow the next call as a probe; IF the probe succeeds, THE circuit_breaker SHALL close; IF the probe fails, THE circuit_breaker SHALL remain open.
6. THE Audit_Interceptor SHALL log all circuit_breaker state transitions (closed → open, open → half-open, half-open → closed, half-open → open) as distinct Audit_Records.

### Requirement 20: Empty Result Handling

**User Story:** As a Skill developer, I want the framework to produce a properly classified, well-formed response when no data is retrieved, so that empty results do not cause undefined behavior or unclassified output.

#### Acceptance Criteria

1. IF `retrieve_data()` returns an empty list, THEN THE BaseSkill SHALL set the High_Water_Mark to the `default_classification`.
2. IF `retrieve_data()` returns an empty list, THEN THE BaseSkill SHALL pass an empty list to `process_data()`, and the Skill developer's implementation SHALL be expected to handle empty input as documented in the Skill_Contract.
3. IF `format_response()` receives an empty or whitespace-only string, THEN THE BaseSkill SHALL generate a default response: "(<default_classification_mark>) No results were found for the given query."
4. THE default empty response SHALL include the Response_Banner, a Portion_Mark, and an empty Sources_Appendix.
5. THE Aggregate_Confidence for an empty result SHALL be 0.0, and a Confidence_Disclaimer SHALL always be prepended for empty results.

### Requirement 21: Orchestrator Hook Adapter Interface (Abstract)

**User Story:** As a platform architect, I want the BaseSkill framework's security policies (classification ceiling, audit logging, portion mark enforcement) to be enforceable at the orchestrator level via pre/post tool hooks, so that guardrails are applied at both the code layer and the platform layer without coupling the framework to any specific orchestrator.

#### Acceptance Criteria

1. THE BaseSkill framework SHALL define an abstract HookAdapter interface with methods: `pre_tool_call(tool_name, arguments, user_context) -> HookDecision`, `post_tool_call(tool_name, result, user_context) -> HookDecision`, and `on_execution_complete(response, user_context) -> HookDecision`.
2. THE HookDecision SHALL be a dataclass containing: `allow` (bool), `reason` (str), `modified_arguments` (optional dict for pre-hook argument rewriting), and `modified_result` (optional for post-hook result filtering).
3. WHEN `pre_tool_call` is invoked, THE HookAdapter SHALL enforce classification ceiling checks by inspecting the tool's expected classification level against `user_context.clearance_level` and SHALL reject calls that would exceed the ceiling by returning a HookDecision with `allow=False`.
4. WHEN `post_tool_call` is invoked, THE HookAdapter SHALL validate that the tool result contains required metadata fields (`classification`, `source`, `confidence`) and SHALL reject results that fail validation by returning a HookDecision with `allow=False`.
5. WHEN `on_execution_complete` is invoked, THE HookAdapter SHALL scan the final response for portion mark compliance and citation completeness.
6. THE HookAdapter interface SHALL NOT import or depend on any orchestrator-specific library.
7. WHEN a hook method returns a HookDecision, THE HookAdapter SHALL log the decision via the Audit_Interceptor with a status of "HOOK_ALLOW" or "HOOK_DENY" including the tool name, the decision reason, and the user_context.user_id.

### Requirement 22: LangGraph Hook Adapter (Reference Implementation)

**User Story:** As a platform engineer, I want a concrete adapter that translates the BaseSkill hook interface into LangGraph callbacks, so that the security guardrails work within a LangGraph-based orchestration pipeline.

#### Acceptance Criteria

1. THE BaseSkill framework SHALL provide a LangGraphHookAdapter class that extends HookAdapter and implements all three hook methods (`pre_tool_call`, `post_tool_call`, `on_execution_complete`).
2. THE LangGraphHookAdapter SHALL translate `pre_tool_call` into a LangGraph `on_tool_start` callback that receives the tool name and input arguments.
3. THE LangGraphHookAdapter SHALL translate `post_tool_call` into a LangGraph `on_tool_end` callback that receives the tool output.
4. THE LangGraphHookAdapter SHALL translate `on_execution_complete` into a LangGraph `on_chain_end` callback.
5. THE LangGraphHookAdapter SHALL carry user_context through the LangGraph callback chain via LangGraph's config/metadata mechanism.
6. THE LangGraphHookAdapter SHALL be importable from an optional subpackage (e.g., `base_skill_framework.adapters.langgraph`) so that the core framework does not require LangGraph as a dependency.
7. IF LangGraph is not installed, THEN importing the core `base_skill_framework` package SHALL NOT raise an ImportError.

### Requirement 23: Orchestrator-Agnostic Security Policy Engine

**User Story:** As a platform architect, I want the security policy logic (ceiling checks, metadata validation, portion mark scanning) to live in a single place that both the BaseSkill and any HookAdapter can call, so that policies are defined once and enforced everywhere.

#### Acceptance Criteria

1. THE BaseSkill framework SHALL provide a SecurityPolicyEngine class that encapsulates all reusable security checks: `check_classification_ceiling(classification, ceiling)`, `validate_fragment_metadata(fragment)`, `validate_portion_marks(text, hwm)`, `validate_citations(text, fragments)`, and `cross_validate_marks_and_citations(text, fragments)`.
2. THE BaseSkill `execute()` pipeline SHALL delegate its validation logic to the SecurityPolicyEngine rather than implementing checks inline.
3. THE HookAdapter interface methods SHALL delegate their validation logic to the same SecurityPolicyEngine instance used by the BaseSkill.
4. WHEN a policy change is made to the SecurityPolicyEngine (e.g., adding a new required metadata field), THE change SHALL automatically apply to both the BaseSkill code path and the orchestrator hook path without requiring duplicate modifications.
5. THE SecurityPolicyEngine SHALL be stateless — all required context (user_context, hwm, fragments) SHALL be passed as method parameters.
6. THE SecurityPolicyEngine SHALL raise the same exception types as the BaseSkill (e.g., ClassificationCeilingExceededError, SourceMissingError) so that error handling is consistent across both enforcement layers.

# Part 3: Prompt Preparation — PR #1006 (aiokafka)

## 3.1.1 Repository Context

aiokafka is an asynchronous Python client library for Apache Kafka that exposes high-level Producer and Consumer APIs built on asyncio. It enables Python applications to produce and consume Kafka messages with an event-driven, non-blocking model suitable for high-throughput, low-latency workloads. Intended users include backend engineers building streaming pipelines, microservice developers using Kafka for eventing, and maintainers of Python projects that require asyncio-compatible Kafka integrations.

The repository contains modules for connection management (conn), cluster metadata (cluster), consumer group coordination (coordinator, assignors), protocol serialization (codec), and tests that validate assignment strategies and protocol compliance. Users expect stable consumer partition assignment semantics, compatibility with common Kafka broker versions, and a clean, well-typed API surface for extending assignors and custom coordination logic. The maintainers aim for readability, robust typing for developer ergonomics (IDE/mypy), and minimal runtime regressions when introducing typing and API clarifications.

This PR focuses on hardening coordinator/assignor internals, adding static typing, and clarifying abstract assignor interfaces. The work must preserve current runtime behavior and test coverage while giving maintainers better type-safety and clearer extension points.

---

## 3.1.2 Pull Request Description

PR #1006 introduces a targeted refactor to the consumer group coordination and assignor code to add static type hints, tighten interfaces, and replace fragile namedtuple patterns with typed NamedTuple definitions. Key changes include:

- Annotating functions, methods, and classmethod signatures across aiokafka/coordinator and aiokafka/cluster with typing primitives (Optional, Set, Dict, Mapping, Iterable).
- Converting various namedtuple usages into typed NamedTuple classes for clearer field semantics.
- Rewriting the SortedSet helper with proper generics, explicit iterator/len/**contains**/pop_last behavior, and predictable KeyError semantics.
- Reworking AbstractPartitionAssignor into a clear ABC pattern: `name` becomes a typed @property with an abstract method, and `assign`, `metadata`, and `on_assignment` are declared as typed @classmethods returning typed structures.
- Small test updates and Makefile entries to include coordinator files and tests/coordinator.

Why: the changes improve maintainability, make API contracts explicit, and allow static analysis (mypy/IDE). Previous behavior relied on implicit runtime conventions (unnamed tuples, untyped dicts), which made extensions and debugging harder. New behavior preserves assignment algorithms but enforces type contracts and slightly changes the abstract API surface (classmethod signatures, typed returns). Attention is required because third‑party custom assignors or subclasses implemented against the old untyped instance-based API may require small updates to conform to classmethod signatures and typed return values.

---

## 3.1.3 Acceptance Criteria

1. When running the full test suite, all existing unit tests (including tests/coordinator/\*) pass without modification failures.
2. When mypy/static typing checks are run (configured to the repo's chosen settings), there are no new type errors in modified files.
3. When a standard consumer group assignment scenario is executed (range, roundrobin, sticky), the assigned partition distribution matches pre-PR behavior for the same inputs (validated by unit tests and a small integration script).
4. The AbstractPartitionAssignor interface compiles and can be subclassed: existing built-in assignors (range, roundrobin, sticky) must implement the new @classmethod signatures and behave identically.
5. SortedSet.pop_last raises KeyError only when empty and returns identical element ordering as before for the same inputs (verified by unit tests).

---

## 3.1.4 Edge Cases

1. Empty topic list or members mapping: assignors must return empty assignments gracefully (no exceptions).
2. Custom third-party assignors that relied on instance methods or untyped return structures: the migration must either keep compatibility shim or provide clear failure modes and a migration guide.
3. SortedSet operations on duplicate keys, empty set pop_last, and iteration during concurrent modifications must behave deterministically; popping from empty SortedSet must raise KeyError.

---

## 3.1.5 Initial Prompt

You are an assistant that will implement and test PR #1006 for the aiokafka repository. Implement the changes exactly as described below, ensuring no functional regressions while introducing static typing and clearer abstract interfaces.

Scope and files to change:

- Focus on aiokafka/coordinator/_, aiokafka/coordinator/assignors/_, aiokafka/coordinator/protocol.py, aiokafka/cluster.py, tests/coordinator/\*, and the Makefile entries that reference coordinator files.
- Add and use typing primitives (from typing import Dict, Iterable, Mapping, Optional, Set, NamedTuple, Sequence) where appropriate.
- Replace ad-hoc namedtuple usages with typed NamedTuple or equivalent typed dataclass-like constructs, keeping field names identical.
- Rework AbstractPartitionAssignor into an ABC that exposes:
  - a typed @property `name: str` declared with @abc.abstractmethod,
  - typed @classmethods `assign(cls, cluster: ClusterMetadata, members: Mapping[str, ConsumerProtocolMemberMetadata]) -> Dict[str, ConsumerProtocolMemberAssignment]`,
  - `metadata(cls, topics: Iterable[str]) -> ConsumerProtocolMemberMetadata`,
  - `on_assignment(cls, assignment: ConsumerProtocolMemberAssignment) -> None`.
- Ensure SortedSet is rewritten generically: iteration, len, `__contains__`, and `pop_last()` must be typed and raise KeyError when empty. Preserve element ordering semantics.
- Annotate ClusterMetadata.partitions_for_topic(topic: str) -> Optional[Set[int]] without changing observed behavior.

Acceptance and tests:

- Ensure the full unit test suite passes. Add or update tests in tests/coordinator to assert typed structures where necessary but do not alter behavioral expectations (assignment outputs must remain the same).
- Add mypy/typing checks in CI if the repo uses type checks; if not, ensure local checks pass (`mypy <files>`).
- Update Makefile FORMATTED_AREAS to include aiokafka/coordinator and tests/coordinator so code formatters and test targets pick up changes.

Edge cases to specifically test and document:

- Empty members mapping or empty topic list returns empty assignments.
- Custom assignor subclasses: create a minimal example subclassing the new AbstractPartitionAssignor and ensure instantiation/use still works; if not, add a short compatibility note in the PR description.
- SortedSet boundary behavior: verify pop_last raises KeyError on empty set and that duplicates and ordering behave as before.

Deliverables:

- Modified source files implementing typing and interfaces.
- Updated or added unit tests validating behavior and edge cases.
- A short PR description summarizing changes, migration notes for third-party assignors, and CI/test verification steps.
- A checklist demonstrating test run results and mypy (or equivalent) status.

Perform changes incrementally and run tests frequently. If a change breaks third-party compatibility, either add a compatibility shim or document migration steps clearly in the PR

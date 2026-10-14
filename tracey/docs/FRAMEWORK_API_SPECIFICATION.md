# Experiment framework API specification

Version: 0.1. Date: 13 October 2026.
Status: unaccepted draft; execution is not implemented.
Independent review found unresolved admission, grouped-write, receipt, ambiguity and scoring issues.
Do not implement this draft until those issues are resolved.

## Authority and scope

This document defines the experiment framework inputs and outputs.
It extends the [mapping API](API_SPECIFICATION.md) for experimental composition.
The [architecture](ARCHITECTURE_SPECIFICATION.md) defines the implementation owners.
The [evaluation record](../contracts/evaluation.json) retains the pinned corpus and release gates.

This contract permits named stage adapters for supported operations.
Feature generators, tokenizers and decoder internals remain private behind those adapters.
The mapping API's complete-document rules apply to its complete-document operation.
They do not require every experimental pipeline to produce a complete document map.
Experimental output must not activate production processing or change a personal trace.
Catalogue reconciliation uses a separate host adapter. It must not enter the pure GROBID mapping crate.
Experiments must apply catalogue changes to isolated copies. Production adapters retain Tracey's guarded storage owners.

Python owns experiment control, scoring and reports.
Shared production operations remain in the planned Rust crate.
Adapters may invoke pinned upstream tools for comparison.
No interface requires Docker, a new server or new model training.

## Public interfaces

| Owner | Function | Input | Output |
| --- | --- | --- | --- |
| E1 | `Stage.describe` | None | Stage ports, implementation identity and permitted operations |
| E2 | `Stage.run` | Named artifacts and execution context | Continue, completed task, stopped action or failure |
| E3 | `compile_pipeline` | Definition and stage registry | Validated composition or configuration errors |
| E4 | `run_pipeline` | Validated composition, case and candidate host | One case result and stage evidence |
| E5 | `run_experiment` | Experiment, compositions, cases and experiment host | Ordered trials and preparation evidence |
| E6 | `evaluate` | Trials, separate reference answers and scoring rules | Metrics, counts and exclusions |

E3 must validate metadata without loading model bytes or scanning documents.
E4 must execute the declared order and enforce one shared budget.
E5 must retain failed trials and every preparation operation.
Only E6 may read reference answers.

## Data conventions

The declarations below specify Python interfaces. They are not runtime source files.
Records serialize as JSON objects. Tuples serialize as arrays. Optional fields serialize as `null`.
All declared record fields are required. Unknown fields must cause validation failure.
Maps must have unique keys. Validated records and nested values must remain immutable during execution.

`Digest` means 64 lowercase hexadecimal characters representing SHA-256.
`Identifier` means 1–128 ASCII letters, digits, underscores, periods or hyphens.
Counters must be integers from zero through 9,007,199,254,740,991.
Coordinates must be finite numbers. Durations use monotonic nanoseconds. Byte counts use bytes.
Display names, timestamps and storage paths must not identify processing behavior.

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Iterator, Literal, Mapping, Protocol, TypeAlias, Union

Digest: TypeAlias = str
Identifier: TypeAlias = str
JsonValue: TypeAlias = Union[
    None, bool, int, float, str, list["JsonValue"], dict[str, "JsonValue"]
]
Task: TypeAlias = Literal["citation_lookup", "bibliography"]
Capability: TypeAlias = Literal[
    "local_hit", "pdf_read", "layout_scan", "asset_read",
    "model_load", "inference", "reference_search",
    "catalogue_read", "catalogue_search", "catalogue_write"
]

@dataclass(frozen=True)
class ArtifactRef:
    kind: str
    schema_sha256: Digest
    content_sha256: Digest
    byte_length: int
    document_sha256: Digest | None

@dataclass(frozen=True)
class DocumentRef:
    pdf: ArtifactRef
    page_count: int

@dataclass(frozen=True)
class Rect:
    x: float
    y: float
    width: float
    height: float

@dataclass(frozen=True)
class PageFrame:
    page: int
    media_box: Rect
    crop_box: Rect
    origin: tuple[float, float]
    user_unit_points: float
    rotation_clockwise: int

@dataclass(frozen=True)
class TextSpan:
    source_artifact: ArtifactRef
    start_scalar: int
    end_scalar_exclusive: int

@dataclass(frozen=True)
class PageBox:
    page: int
    bounds: Rect

@dataclass(frozen=True)
class Trigger:
    kind: Literal["tap", "citation_tool", "text_selection"]
    frame: PageFrame
    point: tuple[float, float] | None
    spans: tuple[TextSpan, ...]
    boxes: tuple[PageBox, ...]
    selected_text: str
    context_text: str
    embedded_destination: ArtifactRef | None
    extractor_sha256: Digest
    coordinate_adapter_sha256: Digest

@dataclass(frozen=True)
class CandidateCase:
    case_id: Identifier
    task: Task
    document: DocumentRef
    trigger: Trigger | None
    known_references: ArtifactRef | None
    inputs: Mapping[str, ArtifactRef]

@dataclass(frozen=True)
class CaseRequest:
    case_id: Identifier
    task: Task
    document: DocumentRef
    trigger: Trigger | None
    known_references: ArtifactRef | None

@dataclass(frozen=True)
class PortSpec:
    name: Identifier
    kind: str
    schema_sha256: Digest
    required: bool

@dataclass(frozen=True)
class OperationPin:
    operation_id: Identifier
    interface_version: int
    implementation_sha256: Digest

@dataclass(frozen=True)
class StageDescriptor:
    operation: OperationPin
    config_schema: ArtifactRef
    inputs: tuple[PortSpec, ...]
    outputs: tuple[PortSpec, ...]
    capabilities: tuple[Capability, ...]
    terminal_tasks: tuple[Task, ...]
    model_roles: tuple[str, ...]

@dataclass(frozen=True)
class InputBinding:
    input_port: Identifier
    source_step: Identifier | None
    source_port: Identifier

@dataclass(frozen=True)
class StepSpec:
    step_id: Identifier
    operation: OperationPin
    config: Mapping[str, JsonValue]
    model_profile: ArtifactRef | None
    bindings: tuple[InputBinding, ...]

@dataclass(frozen=True)
class PipelineDefinition:
    schema: int
    pipeline_id: Identifier
    task: Task
    external_inputs: tuple[PortSpec, ...]
    output_schema_sha256: Digest
    steps: tuple[StepSpec, ...]

@dataclass(frozen=True)
class Limits:
    deadline_ns: int
    memory_bytes: int
    work_units: int
    asset_bytes: int
    output_bytes: int

@dataclass(frozen=True)
class ApiError:
    code: str
    step_id: Identifier | None
    field_path: str | None
    artifact: ArtifactRef | None
    message: str

class CompiledPipeline(Protocol):
    @property
    def identity(self) -> Digest: ...
    @property
    def definition(self) -> PipelineDefinition: ...

@dataclass(frozen=True)
class CompileResult:
    pipeline: CompiledPipeline | None
    errors: tuple[ApiError, ...]
```

Artifact kinds must identify versioned contracts, such as `pdf/v1`, `layout/v1` or `reference-set/v1`.
Each kind must bind to one schema and one registered validator.
Model profiles must retain all required bindings from the existing mapping API.
Feature artifacts must bind to their model, tokenizer, encoding, layout and required context identities.
An output schema must specify the source representation and index units.

Artifact references must identify verified bytes. A pathname alone must not identify an artifact.
`inputs["request"]` must contain a `case-request/v1` artifact with the `CaseRequest` payload.
Its fields must equal the corresponding case fields before stage execution.
The candidate host must verify the declared content digest and length before admitting bytes.
Document artifacts must retain the exact PDF digest. Unrelated document artifacts must fail admission.
The PDF artifact must identify its own content digest as `document_sha256`.
Page count must come from admission of those PDF bytes.

Trigger points and boxes use the unrotated PDF coordinate contract from the mapping API.
Page numbering starts at one. Intrinsic rotation must remain in the page frame.
Character spans use Unicode scalar indices, with an exclusive end.
Selected text must reconstruct from its declared spans when spans are supplied.
Nearby context must retain extracted text. It must not contain expected targets or scoring annotations.
An embedded destination is candidate evidence. Its existence alone does not prove a bibliographic citation.

Citation lookup requires a trigger and a known-reference snapshot.
Bibliography extraction requires `trigger=null`.
A tap requires a point. Text selection requires at least one source span and one page box.
A citation-tool action requires a point or a nonempty selection.
The source artifacts must identify original extracted text and its extraction profile.
Missing extraction information must remain unavailable; the host must not invent it.

## Composition validation

`source_step=null` selects a named external input.
Other bindings must select an output from an earlier declared step.
The initial contract supports ordered sequences. It does not support loops, parallel branches or arbitrary scripts.
One port carries one artifact. Collection artifacts contain multiple references, tokens or boxes.

E3 must reject duplicate step IDs, unknown operations, incompatible schemas and invalid configuration.
It must reject missing required ports, unknown bindings and forward references.
It must reject model roles or required context absent from the declared profile.
It must validate configuration against the pinned configuration schema.
It must resolve implementation metadata without constructing eager model parsers.

For citation lookup, the first step extracts text under the tap. The next step checks the citation candidate.
These steps may use only `local_hit`. They must use the existing admitted PDF reader.
Every expensive stage must consume evidence that the gate admitted the action.
The host must enforce this condition even when a step is misconfigured.
Catalogue adapters may query recorded provider responses after admission.
Personal graph edits are not candidate capabilities.

A successful compile result contains one pipeline and no errors.
A failed compile result contains no pipeline and at least one error.
Only E3 may construct a compiled pipeline.
Each composition must declare a compatible terminal output schema before execution.

## Stage execution and outcomes

```python
@dataclass(frozen=True)
class StageRequest:
    step_id: Identifier
    config: Mapping[str, JsonValue]
    model_profile: ArtifactRef | None
    inputs: Mapping[str, ArtifactRef]

@dataclass(frozen=True)
class Usage:
    duration_ns: int
    peak_memory_bytes: int | None
    work_units: int | None
    pdf_bytes_read: int | None
    asset_bytes_read: int | None
    downloaded_bytes: int | None
    model_loads: int | None
    inference_calls: int | None
    document_scans: int | None
    bibliography_searches: int | None
    cache_hits: int | None
    cache_misses: int | None
    provider_requests: int | None
    catalogue_writes: int | None

@dataclass(frozen=True)
class StageResult:
    status: Literal["continue", "complete", "stop", "error", "cancelled"]
    outputs: Mapping[str, ArtifactRef]
    task_output: ArtifactRef | None
    reason: str | None
    error: ApiError | None

class StageContext(Protocol):
    @property
    def remaining_limits(self) -> Limits: ...
    def is_cancelled(self) -> bool: ...
    def read(self, artifact: ArtifactRef, maximum_bytes: int) -> bytes: ...
    def publish(self, kind: str, schema_sha256: Digest,
                content: bytes, document_sha256: Digest | None) -> ArtifactRef: ...
    def consume_work(self, units: int) -> None: ...
    def hit_test(self, document: DocumentRef, trigger: Trigger,
                 maximum_scalars: int) -> Trigger: ...
    def catalogue(self) -> CataloguePort: ...

class Stage(Protocol):
    def describe(self) -> StageDescriptor: ...
    def run(self, request: StageRequest, context: StageContext) -> StageResult: ...

class StageRegistry(Protocol):
    def describe(self, pin: OperationPin) -> StageDescriptor: ...
    def read_metadata(self, artifact: ArtifactRef) -> JsonValue: ...

@dataclass(frozen=True)
class StageEvidence:
    step_id: Identifier
    operation: OperationPin
    config_sha256: Digest
    model_profile: ArtifactRef | None
    inputs: Mapping[str, ArtifactRef]
    result: StageResult
    usage: Usage

@dataclass(frozen=True)
class PipelineRun:
    run_id: Identifier
    case_id: Identifier
    case_sha256: Digest
    document_sha256: Digest
    pipeline_sha256: Digest
    environment_sha256: Digest
    status: Literal["complete", "stopped", "error", "cancelled"]
    task_output: ArtifactRef | None
    reason: str | None
    error: ApiError | None
    stages: tuple[StageEvidence, ...]
    usage: Usage
```

`continue` returns named outputs and no task output, reason or error.
`complete` returns one validated task output and no error or stop reason.
`stop` returns a stable reason and no task output or error.
`error` returns one error and no task output. `cancelled` returns no task output or error.
All terminal results must have an empty named-output map.
E4 must stop after a terminal result. Reaching the sequence end without a terminal result is an error.

Allowed stop reasons are `non_citation` and `explicit_action_required`.
An uncertain ordinary tap must stop with `explicit_action_required`.
A recognized citation with no match is a completed unresolved lookup.
An ambiguous match is a completed ambiguous lookup. It must not select a target automatically.

Stable error codes are `invalid_input`, `unsupported`, `incompatible_model`, `asset_mismatch`, `budget_exceeded`, `deadline_exceeded`, `stage_failure` and `invalid_output`.
Configuration errors use `invalid_definition`, `unknown_operation`, `invalid_configuration`, `missing_input` or `incompatible_schema`.
Messages must contain at most 240 Unicode scalars. They must not contain unbounded tool output.
Runtime exceptions must become `stage_failure` records. Contract errors must not become successful empty outputs.

The host must verify stage identity, input schemas, output schemas and document bindings.
Model bytes must be verified immediately before first use, after gate admission.
No stage may reset the shared deadline, memory, work or byte limits.
Stages must check cancellation between bounded operations.
The host must terminate non-cooperative execution before releasing its resources and recording completion.

The host must measure usage. Stage-provided claims alone must not establish zero expensive work.
An unavailable measurement is `null`; it must not become zero.
Rejected taps require observed zero asset reads, model loads, inference calls, scans, downloads and bibliography searches.
They must also have zero provider requests and catalogue writes.
Peak memory is an observed maximum. It must not be summed across stages.
Whole-run duration must include framework and adapter overhead.

## Task output payloads

```python
@dataclass(frozen=True)
class Reference:
    reference_id: Digest
    source_artifact: ArtifactRef
    source_spans: tuple[TextSpan, ...]
    raw_text: str
    label: str | None
    title: str | None
    authors: tuple[str, ...]
    year: str | None
    raw_identifiers: tuple[str, ...]

@dataclass(frozen=True)
class ReferenceGroup:
    reference_ids: tuple[Digest, ...]

@dataclass(frozen=True)
class LookupOutput:
    document_sha256: Digest
    trigger_sha256: Digest
    status: Literal["linked", "unresolved", "ambiguous"]
    marker_text: str
    marker_spans: tuple[TextSpan, ...]
    marker_boxes: tuple[PageBox, ...]
    references: tuple[Reference, ...]
    groups: tuple[ReferenceGroup, ...]
    reason: str | None
    citation_probabilities: ProbabilityTable
    paper_probabilities: tuple[PaperProbabilityTable, ...]
    reconciliation: tuple[Reconciliation, ...]
    catalogue_writes: tuple[CatalogueWriteResult, ...]

@dataclass(frozen=True)
class BibliographyOutput:
    document_sha256: Digest
    references: tuple[Reference, ...]
```

`LookupOutput` is the `citation-lookup/v1` artifact payload.
`BibliographyOutput` is the `reference-set/v1` artifact payload.
A linked result has exactly one group. An ambiguous result has at least two distinct groups.
An unresolved result has no groups and a reason.
Allowed unresolved reasons are `no_reference`, `unsupported_marker` and `insufficient_evidence`.
Linked and ambiguous results must use `reason=null`.

Each group must contain unique existing reference IDs in bibliography order.
A group can contain multiple references, such as the targets of `[3–5]`.
Reference IDs identify source bibliography entries. They are not catalogue paper IDs.
Raw identifiers must retain extracted spelling. Missing metadata must remain absent.
The existing Tracey resolver owns normalization and canonical paper identity.

Every lookup must retain marker source spans and boxes for the selected PDF.
The selected point or selection must intersect the returned marker evidence.
Recognizing another citation elsewhere in the paragraph must not validate the user's tap.
Text normalization must not replace original source offsets or geometry.
Reference IDs must use the existing versioned document-and-source identity rules.

## Probability, semantic checks and catalogue reconciliation

The full composition must expose these operations as separate stages:

1. Extract text and source evidence under the tap.
2. Check whether the action can refer to a citation.
3. Find candidate bibliography entries in the displayed PDF.
4. Produce a probability table for bibliography target groups.
5. Compare those entries with the known-reference snapshot.
6. Retrieve missing metadata through the catalogue adapter when needed.
7. Compare independent metadata and semantic similarity.
8. Reconcile missing or conflicting records.
9. Return target evidence, candidate tables and reconciliation results.

Stages may reuse the same verified artifacts. They must not repeat extraction without a declared reason.
The PDF bibliography and database reference list are separate evidence sources.
The database list must not redefine citation numbering in the displayed PDF.

```python
@dataclass(frozen=True)
class PaperRecord:
    paper_key: str
    status: Literal["canonical", "provisional"]
    title: str | None
    authors: tuple[str, ...]
    year: str | None
    identifiers: tuple[str, ...]
    abstract: str | None
    evidence: tuple[ArtifactRef, ...]

@dataclass(frozen=True)
class KnownReferences:
    document_sha256: Digest
    source_paper_key: str
    papers: tuple[PaperRecord, ...]
    provider_receipts: tuple[ArtifactRef, ...]

@dataclass(frozen=True)
class ProbabilityRow:
    hypothesis_id: Identifier
    kind: Literal["reference_group", "known_paper", "not_citation", "other_or_missing"]
    reference_ids: tuple[Digest, ...]
    paper_key: str | None
    raw_score: float | None
    probability: float | None

@dataclass(frozen=True)
class ProbabilityTable:
    status: Literal["calibrated", "unavailable"]
    method: ArtifactRef | None
    calibration: ArtifactRef | None
    rows: tuple[ProbabilityRow, ...]
    unavailable_reason: str | None

@dataclass(frozen=True)
class SemanticCheck:
    reference_id: Digest
    paper_key: str
    left_evidence: tuple[ArtifactRef, ...]
    right_evidence: tuple[ArtifactRef, ...]
    profile: ArtifactRef
    method: Literal["embedding_cosine", "pinned_similarity"]
    score: float | None
    decision: Literal["supports", "conflicts", "insufficient_evidence"]
    reason: str | None

@dataclass(frozen=True)
class PaperProbabilityRow:
    hypothesis_id: Identifier
    paper_key: str | None
    kind: Literal["known_paper", "other_or_missing"]
    identifier_check: Literal["agrees", "conflicts", "absent"]
    semantic: SemanticCheck | None

@dataclass(frozen=True)
class PaperProbabilityTable:
    reference_id: Digest
    known_snapshot_sha256: Digest
    probabilities: ProbabilityTable
    candidates: tuple[PaperProbabilityRow, ...]

@dataclass(frozen=True)
class CatalogueQuery:
    reference: Reference
    known_snapshot_sha256: Digest

@dataclass(frozen=True)
class CatalogueSearchResult:
    reference_id: Digest
    records: tuple[PaperRecord, ...]
    receipts: tuple[ArtifactRef, ...]
    status: Literal["found", "not_found", "unavailable", "conflict"]

@dataclass(frozen=True)
class Reconciliation:
    reference_id: Digest
    status: Literal[
        "existing", "missing_from_reference_list", "missing_paper",
        "metadata_conflict", "unresolved"
    ]
    action: Literal["none", "record_reference", "ensure_paper", "review"]
    paper: PaperRecord | None
    expected_snapshot_sha256: Digest
    idempotency_key: Digest
    evidence: tuple[ArtifactRef, ...]

@dataclass(frozen=True)
class CatalogueWriteResult:
    status: Literal["existing", "created", "reference_recorded", "conflict"]
    paper: PaperRecord | None
    resulting_snapshot: ArtifactRef
    evidence: tuple[ArtifactRef, ...]

class CataloguePort(Protocol):
    def search(self, query: CatalogueQuery) -> CatalogueSearchResult: ...
    def apply(self, decision: Reconciliation) -> CatalogueWriteResult: ...

@dataclass(frozen=True)
class GateEvidence:
    document_sha256: Digest
    trigger_sha256: Digest
    hit: ArtifactRef
    gate: OperationPin

@dataclass(frozen=True)
class ReferenceCandidates:
    reference_id: Digest
    known_snapshot_sha256: Digest
    papers: tuple[PaperRecord, ...]

@dataclass(frozen=True)
class CandidateMatches:
    document_sha256: Digest
    references: tuple[ReferenceCandidates, ...]
```

`known-reference-snapshot/v1` contains the `KnownReferences` payload.
`hit/v1` contains a `Trigger`. `gate-evidence/v1` contains `GateEvidence`.
`candidate-matches/v1` contains `CandidateMatches`.
Other collection artifacts below contain arrays of the specified record type.
Each paper must retain its metadata source and provider receipts.
The snapshot identifies known references before processing. It is not an expected-answer list.
Snapshot identity fields elsewhere contain this artifact's content digest.
E3 must read only admitted configuration and profile metadata through `read_metadata`.
It must not read model payloads through that method.

Probability rows represent mutually exclusive hypotheses, including grouped citations.
The citation table must include `not_citation` and `other_or_missing` hypotheses.
Calibrated probabilities must be finite, within zero and one, and sum to one within `1e-9`.
The method and calibration records must identify their implementations, inputs and independent evaluation evidence.
Candidate truncation must retain omitted probability mass in `other_or_missing`.
Uncalibrated similarities, softmax scores and token-label marginals must not be presented as calibrated link probabilities.
Unavailable tables require null probabilities and a reason. They must block automatic target selection.
No new GROBID training is required. Probability estimation and calibration remain unselected until verified.

Paper candidate probabilities are conditional on the selected bibliography entry.
They must include the possibility that the correct paper is outside the known snapshot.
Each paper row must identify its probability hypothesis without relying on row order.
Paper candidates must correspond one-to-one with their probability rows through `hypothesis_id`.
Reference-group rows have no paper key. Known-paper rows have one paper key and no reference group.
`other_or_missing` paper rows have `paper_key=null`.
Semantic similarity is a separate measurement. It must not be labelled as probability.
Cosine scores range from minus one through one. Other methods require a pinned range and interpretation.
The semantic profile must pin the model, tokenizer, preprocessing, compared fields and decision thresholds.
Missing abstract text must not become an invented abstract.

Semantic checks must compare independently obtained evidence when they support identity verification.
Comparing a reference with a draft copied from that reference is not independent verification.
A high similarity score must not override conflicting verified identifiers.
Insufficient semantic evidence must remain explicit in the result.
The decision policy must specify thresholds and ambiguity margins before an experiment starts.

If the paper exists but the reference list omits it, record the PDF-supported reference separately.
Do not rewrite the provider's original reference list or numbering.
If the paper is missing, `ensure_paper` must create or reuse one record through the existing storage owner.
Incomplete metadata may create a provisional reference record. It must remain visibly unverified.
Provisional keys use `local-reference:` followed by a source-derived SHA-256 digest.
They must not claim a DOI, arXiv identifier or canonical identity that was not verified.
Canonical merging requires identifier and metadata evidence. Semantic similarity alone must not silently merge records.

Catalogue writes must check the expected snapshot and the idempotency key.
Repeated taps must reuse the same record and reference evidence.
Every mutation must retain its write result, including mutations before a later failure.
Conflicts must return `conflict`, without an unrelated write.
Experiments must use isolated, resettable catalogue copies and recorded search responses.
Production adapters may perform real catalogue operations after identity, document and lifetime checks.
Personal trace addition remains a guarded application operation after the result is accepted.

### Required stage ports

Each operation below uses interface version one and its own implementation pin.
Inputs and outputs are named artifact ports. Their schemas must validate the stated payload types.
Every stage after `citation_gate` also requires the admitted `gate` artifact.
`lookup_result` completes execution. Earlier stages continue unless they stop or fail.

| Operation | Input ports and payloads | Output ports and payloads |
| --- | --- | --- |
| `tap_extract` | `request: CaseRequest` | `hit: Trigger` |
| `citation_gate` | `request: CaseRequest`, `hit: Trigger` | `gate: GateEvidence` |
| `bibliography_lookup` | `request: CaseRequest`, `hit: Trigger` | `bibliography: BibliographyOutput` |
| `citation_probability` | `hit: Trigger`, `bibliography: BibliographyOutput` | `citations: ProbabilityTable` |
| `known_candidates` | `bibliography: BibliographyOutput`, `citations: ProbabilityTable`, `known: KnownReferences` | `candidates: CandidateMatches` |
| `catalogue_retrieve` | `bibliography: BibliographyOutput`, `candidates: CandidateMatches` | `retrieved: tuple[CatalogueSearchResult, ...]` |
| `semantic_verify` | `bibliography: BibliographyOutput`, `candidates: CandidateMatches`, `retrieved: tuple[CatalogueSearchResult, ...]` | `checks: tuple[SemanticCheck, ...]` |
| `paper_probability` | `citations: ProbabilityTable`, `candidates: CandidateMatches`, `retrieved: tuple[CatalogueSearchResult, ...]`, `checks: tuple[SemanticCheck, ...]` | `papers: tuple[PaperProbabilityTable, ...]` |
| `reconcile` | `bibliography: BibliographyOutput`, `papers: tuple[PaperProbabilityTable, ...]`, `known: KnownReferences` | `decisions: tuple[Reconciliation, ...]` |
| `catalogue_apply` | `decisions: tuple[Reconciliation, ...]` | `writes: tuple[CatalogueWriteResult, ...]` |
| `lookup_result` | `request: CaseRequest`, `hit: Trigger`, `bibliography: BibliographyOutput`, `citations: ProbabilityTable`, `papers: tuple[PaperProbabilityTable, ...]`, `decisions: tuple[Reconciliation, ...]`, `writes: tuple[CatalogueWriteResult, ...]` | `task_output: LookupOutput` |

`gate-evidence/v1` is an admission permit, not proof of a correct reference match.
The host must accept permits only from the declared gate step for the current action and PDF.
`catalogue_retrieve` must return recorded `found` results for known papers without a new provider request.
`catalogue_apply` must skip `none` and `review` actions without a write.
An experimental rules matcher may replace candidate generation while preserving the declared output contracts.
Fulltext models may require complete document context even when their requested output covers one selected passage.

## Experiment inputs and execution services

```python
@dataclass(frozen=True)
class TrialSpec:
    trial_id: Identifier
    case_id: Identifier
    pipeline_sha256: Digest
    phase: Literal["prepare", "warmup", "measure"]
    condition: Literal["first_use", "installed_cold", "warm_compute", "cached_result"]
    cache_policy: ArtifactRef
    limits: Limits
    seed: int

@dataclass(frozen=True)
class ExperimentSpec:
    schema: int
    experiment_id: Identifier
    hypothesis: str
    dataset_manifest: ArtifactRef
    split: Identifier
    pipeline_sha256s: tuple[Digest, ...]
    environment_requirement: ArtifactRef
    metric_spec: ArtifactRef
    trials: tuple[TrialSpec, ...]

class CandidateHost(Protocol):
    def describe_environment(self) -> ArtifactRef: ...
    def invoke(self, step: StepSpec, request: StageRequest,
               limits: Limits) -> tuple[StageResult, Usage]: ...
    def is_cancelled(self) -> bool: ...

class ExperimentHost(Protocol):
    def candidate_host(self, trial: TrialSpec) -> CandidateHost: ...
    def prepare(self, trial: TrialSpec) -> tuple[ArtifactRef, Usage]: ...
    def release(self, trial: TrialSpec) -> None: ...
    def is_cancelled(self) -> bool: ...

@dataclass(frozen=True)
class TrialRecord:
    trial: TrialSpec
    preparation: ArtifactRef
    preparation_usage: Usage
    run: PipelineRun | None
    error: ApiError | None

@dataclass(frozen=True)
class ExperimentRun:
    experiment_sha256: Digest
    spec: ExperimentSpec
    pipeline_definitions: Mapping[Digest, PipelineDefinition]
    dataset_manifest: ArtifactRef
    environment: ArtifactRef
    trials: tuple[TrialRecord, ...]
    status: Literal["complete", "cancelled", "error"]
    unstarted_trial_ids: tuple[Identifier, ...]
    error: ApiError | None
```

Trial order is explicit. The runner must not generate a hidden random order or repetition schedule.
Repeated trials specify their own IDs, seeds, conditions and limits.
Warm-up trials remain in the raw record. Metric summaries exclude their durations.
Preparation trials produce preparation evidence and no pipeline run.
Measured and warm-up trials require a run or a preparation error.
The host must release each trial's resources, including failed or cancelled trials.

The dataset manifest must list case IDs, input artifact digests, split membership and corpus identifiers.
It must contain no reference answers or expected citation targets.
Duplicate case IDs or missing case inputs must fail validation before execution.
Every trial must select a listed case and compiled pipeline.
Missing cases must not be silently skipped.

Cache policies must specify `assets`, `models`, `layout`, `bibliography`, `references`, `gate` and `lookup` states.
Each state must declare `absent`, `disk`, `resident` or `disabled` and its reset or priming operations.
The preparation record must identify observed states, resets, priming costs and failures.
First use includes required asset transfer. Installed cold processing starts without resident models or document analysis.
Warm compute must disable the final lookup cache. Cached-result trials require a verified primed lookup result.
Cache entries must include exact document, pipeline, schema and processing identities.
One pipeline must not reuse another pipeline's computed results.
Verified unchanged asset bytes may be shared only when the experiment declares that condition.

Environment records must identify device, operating system, architecture, tool versions, builds, timer and memory measurement method.
Candidate hosts receive only admitted candidate artifacts and their declared capabilities.
They must not receive dataset answers, evaluation services or personal storage.
Catalogue access must use only the declared `CataloguePort` after admission.
`invoke` must validate the operation pin and enforce the effective limits.
Stage construction must not load model bytes before the stage requires them.
Unexpected resource or network activity must be recorded and fail the applicable acceptance check.

## Scoring inputs and outputs

```python
@dataclass(frozen=True)
class GoldCase:
    case_id: Identifier
    input_case_sha256: Digest
    coverage: Literal["positive", "negative", "unscorable"]
    reference_annotations: ArtifactRef | None
    geometry_annotations: ArtifactRef | None
    catalogue_annotations: ArtifactRef | None
    reason: str | None

@dataclass(frozen=True)
class MetricSpec:
    schema: int
    task: Task
    output_schema_sha256: Digest
    scorer: OperationPin
    reference_alignment: ArtifactRef
    field_normalization: ArtifactRef
    geometry_policy: ArtifactRef | None
    identity_policy: ArtifactRef | None
    metrics: tuple[str, ...]
    percentile: Literal["nearest_rank"]

@dataclass(frozen=True)
class MetricValue:
    name: str
    value: float | None
    numerator: int | None
    denominator: int | None
    sample_count: int
    unavailable_reason: str | None

@dataclass(frozen=True)
class ScoreGroup:
    pipeline_sha256: Digest
    corpus: Identifier
    condition: str
    metrics: tuple[MetricValue, ...]
    counts: Mapping[str, int]

@dataclass(frozen=True)
class Exclusion:
    case_id: Identifier
    metric: str
    reason: str

@dataclass(frozen=True)
class EvaluationReport:
    experiment_sha256: Digest
    run_artifact: ArtifactRef
    gold_manifest: ArtifactRef
    metric_spec_sha256: Digest
    groups: tuple[ScoreGroup, ...]
    exclusions: tuple[Exclusion, ...]
    upstream_agreement: ArtifactRef | None

class EvaluationInputError(Exception):
    def __init__(self, error: ApiError) -> None: ...

class EvaluationHost(Protocol):
    def read(self, artifact: ArtifactRef, maximum_bytes: int) -> bytes: ...
```

Positive citation annotations must identify the marker and acceptable bibliography target groups.
They must provide source-reference alignment independent of pipeline-specific reference IDs.
Negative annotations must explicitly establish that the action is not a bibliographic citation.
Unscorable annotations require a reason. Missing annotations must become unscorable, never negative.
Geometry annotations must identify the exact PDF, page frames, reviewed boxes and annotation source.
The scorer must reject answers bound to different case or PDF bytes.

Allowed metric keys are defined in the table below.
Each metric must declare a compatible task and required annotations.
Unsupported outputs must be reported as unavailable. Empty output must not simulate unsupported capability.

Reference alignment must map candidate entries to annotated source entries before target scoring.
Target scoring uses reference sets. A wrong target contributes a false positive and a missed expected target.
Rejected, unresolved, ambiguous, failed and cancelled positive cases remain in recall denominators.
Group accuracy requires an exact acceptable target set. Partial range matches must not count as exact success.
False activation means returning a linked lookup for an annotated negative tap.
Negative expensive-work rate measures negative taps with observed expensive work, including later rejected taps.
Unknown work counters make that measurement unavailable; they must not establish a zero-work pass.

Reports must retain total, admitted, rejected, positive, negative, unscorable, failed, cancelled and unsupported case counts.
They must retain target true-positive, false-positive and false-negative counts.
Zero denominators produce unavailable values with reasons.
F1 is `2*TP/(2*TP+FP+FN)` when its denominator is positive.
Nearest-rank percentile selects sorted sample `ceil(p*n)-1`, using zero-based indices.
Latency summaries use measured trials only and must report their sample count.
Publish each corpus and cache condition separately. Do not combine different measurement methods silently.
Upstream agreement is separate from correctness against independent reference answers.
Probability calibration requires uniquely annotated hypotheses and an independently identified evaluation split.
Reconciliation scoring must distinguish missing reference edges, missing papers, conflicting metadata and genuine non-citations.
Adding duplicates or merging distinct papers must count as errors.

| Metric key | Definition |
| --- | --- |
| `target_precision` | TP divided by TP plus FP. |
| `target_recall` | TP divided by TP plus FN. |
| `target_f1` | The F1 formula above. |
| `group_exact_accuracy` | Exact accepted groups divided by annotated positive cases. |
| `field_accuracy` | Correct fields divided by independently annotated fields. |
| `false_activation_rate` | Linked negative taps divided by annotated negative taps. |
| `missed_citation_rate` | Positive cases without an exact accepted group divided by annotated positive cases. |
| `negative_expensive_work_rate` | Negative taps with expensive work divided by negative taps with complete work measurements. |
| `geometry_iou` | Intersection over union under the pinned reviewed-geometry policy. |
| `latency_p50_ns` | Nearest-rank median of measured successful trial durations. |
| `latency_p95_ns` | Nearest-rank 95th percentile of measured successful trial durations. |
| `peak_memory_bytes` | Maximum measured memory across the applicable measured trials. |
| `probability_brier` | Mean sum of squared class probability errors on uniquely labelled cases. |
| `reconciliation_accuracy` | Correct reconciliation decisions divided by annotated reconciliation cases. |
| `duplicate_rate` | Unwanted duplicate additions divided by annotated missing-paper addition attempts. |

Latency reports must retain failed-trial durations and counts separately.
Probability metrics must exclude calibration-fit data and record excluded or unavailable probabilities.
Unknown work measurements must reduce reported measurement coverage, not the known negative-tap count.

## Function signatures and identity

```python
def compile_pipeline(
    definition: PipelineDefinition, registry: StageRegistry
) -> CompileResult: ...

def run_pipeline(
    pipeline: CompiledPipeline, case: CandidateCase,
    limits: Limits, host: CandidateHost
) -> PipelineRun: ...

def run_experiment(
    spec: ExperimentSpec,
    pipelines: Mapping[Digest, CompiledPipeline],
    cases: Mapping[Identifier, CandidateCase],
    host: ExperimentHost
) -> ExperimentRun: ...

def evaluate(
    run: ExperimentRun, run_artifact: ArtifactRef,
    gold_manifest: ArtifactRef, gold: Mapping[Identifier, GoldCase],
    metrics: MetricSpec, host: EvaluationHost
) -> EvaluationReport: ...
```

Pipeline identity must hash the complete resolved definition, ordered stage descriptors, effective settings and model profile references.
Use domain `gf-experiment-pipeline-v1` and canonical JSON from [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785).
Reject duplicate keys, non-finite values and unsupported integer ranges before canonicalization.
Experiment, case and metric identities use domains `gf-experiment-v1`, `gf-case-v1` and `gf-metric-v1`, respectively.
Encode the ASCII domain, one NUL byte and canonical record bytes before SHA-256.
Changing operation order, implementation, configuration or model resources must change pipeline identity.
This identity does not replace the production runtime or map profile identities.
No record may claim that expected asset hashes establish verified inference compatibility.

E5 must validate the experiment, schedule and all selected compositions before starting a trial.
Invalid experiments return `status=error`, no trials and all declared trial IDs as unstarted.
E6 must validate the recorded run digest, gold bindings, scorer identity and output schemas before scoring.
Invalid scoring inputs must raise a typed `EvaluationInputError` carrying an `ApiError`.
E6 must never return a report for invalid scoring inputs.

## Acceptance checks

1. Reject incompatible ports and model context before loading assets.
2. Reject duplicate IDs, unknown operations, forward bindings and invalid settings.
3. Reject ordinary taps with zero measured expensive work.
4. Stop ambiguous results without automatic paper selection.
5. Reject stale document, extraction, schema and profile identities.
6. Preserve range groups, source text and original PDF positions.
7. Block reference answers and application state from candidate stages.
8. Record every preparation, warm-up, measured, failed and unstarted trial.
9. Separate cold, warm and cached measurements with their preparation costs.
10. Keep failed positive cases in recall denominators.
11. Keep missing annotations separate from negative labels.
12. Report incompatible metrics and absent measurements as unavailable.
13. Repeat identity checks after each behavior-affecting change.
14. Compare raw stage evidence before explaining an accuracy difference.

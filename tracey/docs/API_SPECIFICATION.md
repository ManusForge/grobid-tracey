# Citation mapping API specification

Status: proposed contract; no API in this document is implemented.
Scope: `tracey/crates/grobid-fork` inside this fork.
Baseline: GROBID `0.9.2-SNAPSHOT`, commit `c71ebff51e4d2a8257885458daeedede82698d43`.
The crate must have no Tracey dependency.

The [experiment framework API](FRAMEWORK_API_SPECIFICATION.md) defines experimental composition and evaluation.
Its named stage adapters extend the experiment boundary; model-specific implementations remain private.
Complete-document rules below apply to `map_document`, not every experimental output.
No experimental result establishes production acceptance.

Read [requirements](SPECIFICATION.md), [architecture](ARCHITECTURE_SPECIFICATION.md), and [model compatibility](MODEL_COMPATIBILITY.md) with this contract.
These documents must agree before implementation starts.
The Rust declarations below specify interfaces; they are not source files.
The initial asset inventory is unverified and must not produce a runtime profile until dependency closure passes verification.
That candidate inventory must record `closure_verified=false` and `runtime_profile_id=null`.
The candidate JSON record is inventory evidence, not a serialized `ModelManifest` accepted by F3.

## Terms and ownership

A host is the application that supplies layout and model assets.
A source ID identifies one supplied layout object inside one document.
A span identifies a contiguous range of supplied tokens.
A profile is a versioned record that describes processing behavior.
A profile identity is the SHA-256 digest of that record's canonical encoding.
A manifest is the complete list of required assets and their compatibility information.

The host must retain ownership of PDF bytes, extraction, storage, network access, and publication.
The fork must return document analysis without catalogue identities or application state.
`tracey-documents` must retain `Receipt`, `CitationMap`, the map profile, and the citation cache.
Its PDFium adapter must own the final page transform.
Its existing identifier normalizer and catalogue resolver must process raw identifiers after mapping.
The fork must not accept a `Receipt`, emit a `CitationMap`, or introduce another PDF engine.

## Public function owners

| Owner | Interface | Responsibility |
| --- | --- | --- |
| F1 | `map_document` | Run the complete bounded mapping operation. |
| F2 | `validate_layout` | Validate layout before feature generation. |
| F3 | `verify_assets` | Read, verify, and bind the required assets. |
| F4 | `runtime_profile` | Compute the fork processing identity. |
| F5 | `AssetSource::read` | Supply one bounded asset read. |
| F6 | `SequenceDecoder::decode` | Decode one validated feature sequence. |
| F7 | `SequenceDecoder::identity` | Declare the exact inference implementation. |
| F8 | `Cancellation::is_cancelled` | Report a host cancellation request. |

Segmentation, feature extraction, parsing, matching, and box assembly must remain private stages.
These stages must use immutable inputs and explicit outputs.
The orchestrator must not contain model-specific feature rules.

## Proposed entry points

```rust
pub fn map_document(
    input: &LayoutDocument,
    assets: &VerifiedAssets,
    decoder: &mut dyn SequenceDecoder,
    cancel: &dyn Cancellation,
    limits: &RunLimits,
) -> Result<MappingResult, MappingError>;

pub fn validate_layout<'a>(
    input: &'a LayoutDocument,
    limits: &RunLimits,
    cancel: &dyn Cancellation,
) -> Result<ValidatedLayout<'a>, MappingError>;

pub fn verify_assets(
    manifest: &ModelManifest,
    source: &mut dyn AssetSource,
    cancel: &dyn Cancellation,
    limits: &RunLimits,
) -> Result<VerifiedAssets, MappingError>;

pub fn runtime_profile(
    assets: &VerifiedAssets,
    decoder: &dyn SequenceDecoder,
) -> Result<RuntimeProfile, MappingError>;

pub trait AssetSource {
    fn read(&mut self, id: &AssetId, offset: u64, max_bytes: usize)
        -> Result<Vec<u8>, AssetReadError>;
}
pub trait SequenceDecoder {
    fn identity(&self) -> DecoderIdentity;
    fn decode(&mut self, request: &DecodeRequest<'_>,
        cancel: &dyn Cancellation, budget: &mut DecodeBudget)
        -> Result<DecodedSequence, DecodeError>;
}
pub trait Cancellation { fn is_cancelled(&self) -> bool; }
```

F1 must call F2 before any feature generation or decoder call.
F1 must compare the decoder identity with the verified inference pin before processing.
F3 must reject missing, extra, oversized, truncated, or hash-mismatched required asset content.
F3 must check cancellation between reads and use checked size arithmetic.

F5 must return at most `max_bytes` bytes; an empty result means end of asset.
F3 must verify the declared end using a bounded read after the expected asset length.
`VerifiedAssets` and `ValidatedLayout` must have private constructors and immutable contents.
F1 must apply the current limits even when assets were verified with different limits.
F4 must reject an incompatible decoder instead of producing a profile.

## Layout input

```rust
pub struct DocumentKey { pub pdf_sha256: [u8; 32], pub page_count: u32 }
pub struct SourceId(pub u64);
pub struct TokenRange { pub start: u32, pub end_exclusive: u32 }
pub struct Rect { pub x: f64, pub y: f64, pub width: f64, pub height: f64 }
pub struct PageFrame {
    pub page: u32,
    pub media_box: Rect,
    pub crop_box: Rect,
    pub origin: [f64; 2],
    pub user_unit_points: f64,
    pub rotation_clockwise: u16,
}
pub struct FontStyle {
    pub id: SourceId, pub family: Option<String>, pub size: Option<f64>,
    pub bold: Option<bool>, pub italic: Option<bool>,
    pub superscript: Option<bool>, pub subscript: Option<bool>,
}
pub struct LayoutToken {
    pub id: SourceId, pub page: u32, pub text: String,
    pub bounds: Rect, pub style: SourceId, pub reading_order: u32,
}
pub struct LayoutLine {
    pub id: SourceId, pub page: u32, pub tokens: TokenRange,
    pub bounds: Rect, pub reading_order: u32,
}
pub struct LayoutBlock {
    pub id: SourceId, pub page: u32, pub lines: Vec<SourceId>,
    pub bounds: Rect, pub reading_order: u32,
}
pub enum GraphicKind { Unknown, Bitmap, Vector, VectorBox }
pub struct GraphicBox {
    pub id: SourceId, pub page: u32, pub kind: GraphicKind,
    pub bounds: Rect, pub reading_order: u32,
}
pub struct LayoutDocument {
    pub layout_contract: u32, pub key: DocumentKey, pub pages: Vec<PageFrame>,
    pub tokens: Vec<LayoutToken>, pub lines: Vec<LayoutLine>,
    pub blocks: Vec<LayoutBlock>, pub styles: Vec<FontStyle>,
    pub graphics: Option<Vec<GraphicBox>>,
    pub complete_citation_scope: bool,
}
```

Page numbers must start at one and cover every page in `DocumentKey` exactly once.
Coordinates must use the supplied unrotated PDF frame, with positive X rightward and positive Y upward.
`origin` must specify the coordinate offset from PDF default user space.
The host must preserve MediaBox, CropBox, rotation, and PDF user-unit scale without applying display rotation to token boxes.
Rotation must be 0, 90, 180, or 270 degrees.

The fork must preserve page frames unchanged in its output.
Only the host adapter may convert output boxes to normalized reader coordinates.

F2 must reject non-finite coordinates, nonpositive page sizes, invalid page references, and boxes outside the declared MediaBox.
Delimiter-only tokens may retain zero-area rectangles; visible tokens and graphic boxes must have positive area.
F2 must reject duplicate source IDs, unknown style IDs, invalid spans, repeated ownership, and inconsistent line or block membership.
Each token must belong to exactly one line; each line must belong to exactly one block.
Token ranges must follow the supplied token vector and must not cross pages.

Reading-order values must establish a unique order within each object class.
F2 must reject NUL characters and unsupported layout contract versions at the host boundary.
The host must verify `DocumentKey` against the exact PDF receipt before calling F1.
The fork cannot verify the PDF hash from layout alone.

The host must supply ordered graphic boxes and their upstream graphic types for bitmap-proximity features.
`graphics=None` means unavailable information; an empty supplied vector means the document has no graphic objects.
The host must supply complete font, style, and superscript information required by the selected feature profile.
Missing required feature information must fail with `UnsupportedLayoutProfile`; the fork must not substitute zero values.
Omitting an output stage must not remove an input feature needed by a retained model.
For font fields, `None` means unavailable; `Some(false)` means an observed false value.

`LayoutToken.text` must retain original extracted text, including delimiters, without normalization.
Concatenating token text in source order must reconstruct the extracted source exactly.
The fork must not infer literal spaces or line breaks from style or layout flags.
Normalized feature text must be a private copy; source evidence must remain unchanged.

F1 must cover citations in abstracts, body paragraphs, lists, notes, captions, annexes, and appendices.
The host must supply all extracted text from these regions, including regions without recognized citations.
`complete_citation_scope` must attest this extraction coverage; a false value must fail admission.
Regions absent from the original PDF must not count as missing input.
Bibliography entry labels must not become citation callouts.
Missing required input regions or unsupported required region processing must cause `UnsupportedDocumentScope` without a `MappingResult`.

## Assets and decoding

```rust
pub struct AssetId(pub String);
pub struct AssetEntry {
    pub id: AssetId, pub sha256: [u8; 32], pub byte_length: u64,
}
pub enum ModelKind {
    Segmentation, ReferenceSegmenter, Citation, NameCitation, Date, Fulltext,
    Header, NameHeader, AffiliationAddress, Figure, Table,
}
pub enum ResourceRole { Lexicon, Tokenization, Normalization, Matching }
pub struct ResourceBinding {
    pub role: ResourceRole, pub assets: Vec<AssetId>,
    pub lookup_contract_sha256: [u8; 32],
}
pub struct ModelBinding {
    pub kind: ModelKind, pub model: AssetId, pub flavor: String,
    pub label_order: Vec<String>,
    pub feature_contract_sha256: [u8; 32],
    pub tokenizer_contract_sha256: [u8; 32],
    pub resource_indices: Vec<u32>,
    pub feature_boundary_encoding_sha256: [u8; 32],
}
pub struct DecoderIdentity {
    pub implementation: String, pub revision: String,
    pub build_sha256: [u8; 32], pub inference_contract_sha256: [u8; 32],
}
pub struct ModelManifest {
    pub schema: u32, pub upstream_commit: String,
    pub assets: Vec<AssetEntry>, pub bindings: Vec<ModelBinding>,
    pub resources: Vec<ResourceBinding>,
    pub inference_pin: DecoderIdentity, pub hard_limits: RunLimits,
    pub closure_verified: bool,
}
pub struct FeatureValue { pub encoded_bytes: Vec<u8> }
pub struct FeatureRow { pub source_tokens: Option<TokenRange>, pub values: Vec<FeatureValue> }
pub struct FeatureBatch {
    pub schema_sha256: [u8; 32], pub tokenizer_sha256: [u8; 32],
    pub boundary_encoding_sha256: [u8; 32], pub rows: Vec<FeatureRow>,
    pub sequence_ends_exclusive: Vec<u32>,
}
pub struct DecodeRequest<'a> {
    pub model: &'a VerifiedModel, pub features: &'a FeatureBatch,
}
pub struct VerifiedModel { /* private binding, immutable model bytes, and verified resources */ }
impl VerifiedModel {
    pub fn binding(&self) -> &ModelBinding;
    pub fn model_bytes(&self) -> &[u8];
    pub fn resources(&self) -> &[ResourceBinding];
}
pub struct DecodedSequence { pub labels: Vec<String> }
```

The feature contract must specify column order, value encoding, whitespace, boundary rows, and every feature rule.
The first six variants must bind to `segmentation`, `reference-segmenter`, `citation`, `name-citation`, `date`, and `fulltext`, respectively.
`name-citation` must use upstream `name/citation/model.wapiti`; unknown model roles must fail verification.
The auxiliary aliases must be `header`, `name-header`, `affiliation-address`, `figure`, and `table`, respectively.
Auxiliary roles may activate only when verified dependency closure proves their necessity and supplies their complete bindings.
The six candidate models are a minimum inventory; they do not establish complete citation-region coverage.

Feature rows must retain their source-token ranges and explicit sequence boundaries.
Boundary rows must have no source span; all other rows must have a valid span.
Only `VerifiedAssets` may create a `VerifiedModel`; external code must not construct or mutate it.
Its read-only view must expose the trusted model bytes, ordered labels, feature schema, tokenizer contract, boundary encoding, and resource bindings.

F6 must compare supplied feature, tokenizer, and boundary digests with this trusted binding before decoding.
F6 must validate feature column counts and boundary rows against the trusted feature contract.
The tokenizer contract must specify token boundaries, Unicode behavior, and reconstruction rules.

Feature values must be bounded encoded bytes because UTF-16 slicing and JNI modified UTF-8 can produce nonstandard UTF-8.
Their encoded lengths must debit feature and memory budgets before allocation.
The verified boundary-encoding contract must define exact bytes and separators at the Java-to-C baseline boundary.

Each lexicon must be an individually hashed asset with a defined role and order.
Ordered resource bindings must cover lexicons, tokenization, normalization, and matching dependencies.
Each binding must preserve asset order and pin its lookup rules through `lookup_contract_sha256`.

Model resource indices must select ordered manifest bindings; shared private stages must use the same verified resource table.
The manifest must list exactly the assets required by the selected citation pipeline.
F3 must reject duplicate asset IDs, missing bindings, invalid resource indices, unknown flavors, and unsupported manifest versions.
F3 must reject an unverified dependency closure instead of issuing `VerifiedAssets`.
The caller must select a trusted manifest; content hashes do not establish manifest authenticity.

The bibliography stages must derive known-reference and first-author flags before fulltext feature generation.
Fulltext features must retain bitmap proximity and superscript behavior from upstream.
Context-dependent normalization must occur exactly once at the upstream-equivalent stage.
Feature rules must preserve Java UTF-16 behavior and distinguish float32 integer-bin truncation from float64 arithmetic.
The [compatibility contract](MODEL_COMPATIBILITY.md) must specify these precision and normalization rules before decoder admission.

F6 must reuse established C Wapiti inference through a narrow adapter.
Native and WebAssembly adaptation remain unproven until compatibility and resource tests pass.
The contract does not authorize a replacement generic machine-learning engine.
F6 must emit exactly one allowed label per supplied feature row, excluding contract-defined boundary rows.

The orchestrator must reject extra labels, missing labels, and unsupported model formats.
The adapter must honor cancellation and the supplied decode budget inside inference.
An adapter without bounded cancellation or memory behavior must fail admission.
F6 must not return fabricated confidence scores.

## Mapping output

```rust
pub struct ReferenceId(pub [u8; 32]);
pub struct SourceOccurrenceId(pub [u8; 32]);
pub struct RuntimeProfile(pub [u8; 32]);
pub enum IdentifierKind { Doi, Arxiv }
pub struct RawIdentifier { pub kind: IdentifierKind, pub text: String }
pub struct ReferenceRecord {
    pub id: ReferenceId, pub source_tokens: Vec<TokenRange>,
    pub raw_text: String, pub label: Option<String>, pub title: Option<String>,
    pub authors: Vec<String>, pub year: Option<String>,
    pub raw_identifiers: Vec<RawIdentifier>,
}
pub struct PageBox { pub page: u32, pub bounds: Rect }
pub enum UnresolvedReason { NoReference, UnsupportedMarker, InsufficientEvidence }
pub enum MatchDecision {
    Linked { references: Vec<ReferenceId> },
    Unresolved { reason: UnresolvedReason },
    Ambiguous { alternatives: Vec<Vec<ReferenceId>> },
}
pub struct CitationOccurrence {
    pub source_id: SourceOccurrenceId, pub marker: String,
    pub source_tokens: Vec<TokenRange>, pub boxes: Vec<PageBox>,
    pub decision: MatchDecision,
}
pub struct MappingResult {
    pub document: DocumentKey, pub runtime_profile: RuntimeProfile,
    pub pages: Vec<PageFrame>, pub references: Vec<ReferenceRecord>,
    pub occurrences: Vec<CitationOccurrence>,
}
```

Every occurrence must contain its original marker, source spans, and at least one page box.
The fork must preserve multiple boxes for markers that cross lines.
`Linked` must identify one supported reference group; a group may contain multiple references.
`Ambiguous` must preserve at least two distinct candidate groups without selecting one.
Each candidate group must contain unique, existing reference IDs in bibliography order.

Unresolved and ambiguous occurrences must remain visible in the analysis result.
Missing parsed fields must remain absent; the fork must not invent bibliographic metadata.
DOI and arXiv text must remain raw extraction evidence, including its original spelling.
The parent must normalize this text through its existing normalizer before catalogue lookup.

`RuntimeProfile` must carry the fork profile identity; it must not represent the profile record itself.
F4 must hash the complete manifest, fork algorithm revision, contract versions, and exact decoder identity.
This includes every binding field, model role, flavor, label order, resource role, resource order, asset hash, and encoding digest.
The canonical profile must start with the `gf-runtime-v1` domain and a versioned schema identifier.
Fields must follow schema order; field names, strings, and lists must use unsigned 64-bit big-endian length prefixes.
Integers must use fixed-width big-endian encoding; booleans must use one byte; digests must use their fixed 32 bytes.

Nested records must follow the same rules; resource and model lists must retain declared order.
Optional fields must encode a presence byte before their value; no behavior-affecting field may remain unhashed.
The parent must compose its full map profile from this runtime profile, extractor identity, coordinate adapter identity, and output schema identity.
The fork must not create a second full map profile or cache key.

Reference IDs and source occurrence IDs must derive from versioned canonical source spans and their document key.
Output ordering must follow page order, reading order, source spans, and source IDs as explicit tie breakers.
Equal admitted inputs, verified assets, and decoder identity must produce identical successful output bytes under the specified canonical encoding.

The parent must derive final occurrence IDs from the full map profile, PDF hash, and source occurrence ID.
The persisted occurrence ID must use the parent's `gf1` prefix and canonical identity digest.
Saved selections must retain that complete identity; regenerated `c0` must never rebind a saved selection.
Legacy `cN` identifiers must remain historical identifiers and must never be reinterpreted against a regenerated map.

## Limits, errors, and publication

```rust
pub struct RunLimits {
    pub pages: u32, pub tokens: u64, pub lines: u64, pub blocks: u64,
    pub styles: u64, pub graphics: u64, pub text_bytes: u64, pub assets: u64,
    pub asset_bytes: u64, pub total_asset_bytes: u64, pub feature_bytes: u64,
    pub sequence_tokens: u64, pub references: u64, pub occurrences: u64,
    pub boxes: u64, pub output_bytes: u64, pub work_units: u64, pub memory_bytes: u64,
    pub cancellation_chunk_tokens: u32,
}
pub struct DecodeBudget { pub remaining_work: u64, pub remaining_memory: u64 }
pub enum Stage { Layout, Assets, Features, Decode, Parse, Match, Boxes, Output, Profile }
pub enum ErrorKind {
    InvalidInput, UnsupportedLayoutProfile, UnsupportedDocumentScope, InvalidLimits,
    AssetRead, AssetMismatch, IncompatibleModel,
    IncompatibleDecoder, LimitExceeded, DecodeFailed, InvalidDecoderOutput,
    Cancelled, InternalInvariant,
}
pub struct MappingError { pub kind: ErrorKind, pub stage: Stage, pub context: String }
pub struct AssetReadError { pub context: String }
pub struct DecodeError { pub kind: ErrorKind, pub context: String }
pub struct VerifiedAssets { /* private verified manifest and immutable bytes */ }
pub struct ValidatedLayout<'a> { /* private borrowed LayoutDocument */ }
```

`RunLimits` must contain positive caps for pages, tokens, lines, blocks, styles, graphics, text bytes, assets, and total asset bytes.
It must also cap per-asset bytes, feature bytes, sequence length, references, occurrences, boxes, output bytes, work units, and memory bytes.
The manifest must declare compatible hard ceilings; the effective cap must be the smaller declared limit.
Parent admission must retain the current 40 MiB PDF, 600-page, 4 MiB output, 10,000-reference, and 50,000-occurrence ceilings.
The fork must reject page, output, reference, or occurrence limits above these ceilings.
PDF byte admission must remain in the parent because this API receives layout.

Every counter must use checked arithmetic and reject overflow before allocation or inference.
The implementation must define and test memory accounting for Rust storage and the Wapiti adapter before release.
`DecodeBudget` must debit the shared work and memory budgets; a model call must not reset either budget.
Limits must cause errors, never silent truncation or altered successful matches.

Each error must carry a stable code, stage, and bounded context identifying the affected input or asset.
`AssetReadError` and `DecodeError` must map to these errors without exposing unbounded native diagnostics.
Cancellation must be checked at stage boundaries, between asset reads, and inside bounded token and inference loops.
The host must configure a positive token chunk for cancellation checks; that chunk must also bound private token loops.
The host must enforce the 40-second deadline independently and terminate processing when cooperative cancellation fails.
The acceptance suite must measure cancellation acknowledgement and host termination for each supported adapter.

Any cancelled or failed operation must return no `MappingResult` and must release temporary resources.
The host must publish and cache only a complete result after validating document identity, profile, boxes, and reference links.
The host must never publish a partially completed map.

## Acceptance checks

1. Reject invalid frames, missing feature inputs, duplicate IDs, malformed spans, and oversized inputs before decoder invocation.
2. Reject each missing, corrupted, extra, or incompatible required asset.
3. Compare feature rows and labels with pinned upstream fixtures before accepting Wapiti adaptation.
4. Check grouped, unresolved, ambiguous, repeated, and line-crossing markers without invented scores.
5. Repeat identical runs and compare canonical output bytes.
6. Change each profile component and require a changed profile.
7. Cancel every stage and verify no published result or leaked resource.
8. Exhaust each budget and verify a typed error without truncation.
9. Verify rotated and cropped page boxes through the parent PDFium transform.
10. Regenerate reordered occurrences and verify saved selections cannot bind to new `cN` positions.

# GROBID-fork activity specification

Version: 0.1. Date: 13 October 2026.
Status: specified; runtime acceptance pending.

## Purpose and authority

GROBID-fork must map PDF citation markers to bibliography entries for Tracey.
It must reuse the upstream models and established inference where compatible.
New work must contain the required pipeline adaptation and application boundary.
It must not train replacement models or add unrelated document analysis.

This document controls behavior. The [architecture](ARCHITECTURE_SPECIFICATION.md) controls ownership.
The [API specification](API_SPECIFICATION.md) controls interfaces.
The [compatibility review](MODEL_COMPATIBILITY.md) controls model reuse.
The two contract records control source identities and evaluation inputs.

Write technical prose in ASD-STE100 style. Use the technical terms below consistently.
Use sentences of at most 25 words. Use acceptance instructions of at most 20 words.
Use paragraphs of at most six sentences. These limits do not establish formal language certification.

## Technical terms

| Term | Meaning |
| --- | --- |
| Baseline | The unchanged upstream implementation at the recorded commit. |
| CRF | A conditional random field used for sequence labeling. |
| Model | Trained parameters, labels and feature templates used by inference. |
| Inference | The calculation that gives labels to prepared input. |
| Decode | Run sequence inference with a specified model. |
| Feature | A model input, such as token text, position or font information. |
| Feature schema | The ordered columns and interpretation of model input. |
| Token | A text unit with its source identity and layout information. |
| Tokenize | Divide input into the token units required by the model. |
| Normalize | Apply the recorded upstream text transformations. |
| Lexicon | A versioned dictionary used to prepare model features. |
| Model profile | A versioned record of required models, resources and processing rules. |
| Runtime profile identity | The digest of the verified runtime processing description. |
| Map profile identity | The digest that also identifies PDF extraction, adaptation and output schema. |
| Digest | A SHA-256 value that identifies exact bytes. |
| Citation marker | Text in a paper that refers to a bibliography entry. |
| Occurrence | One source location of a citation marker. |
| Reference | A bibliography entry; it is not a canonical catalogue identity. |
| Group | One marker that names more than one reference. |
| Page frame | Page dimensions, boxes, origin and intrinsic rotation. |
| Adapter | Code that changes one verified interface representation into another. |
| Admit | Accept an input after its required checks pass. |
| Validate | Check an input against its contract. |
| Cache | Keep a disposable result under its complete processing identity. |
| Gold data | Published reference annotations used to score extraction quality. |
| Parity | Agreement under the same recorded inputs and configuration. |
| Gate | A check that must pass before the next stage is accepted. |
| Resource closure | The complete list of files and components required by the selected pipeline. |
| Host generation | A lifetime value that changes when the host replaces or invalidates an operation. |
| Serialize | Encode fields in the specified byte representation. |
| Hash | Calculate a digest from the specified bytes. |
| JDK | Java Development Kit; its version is part of the baseline environment. |
| JNI | Java Native Interface; the bridge used by the upstream native decoder. |
| CJK | Chinese, Japanese and Korean language processing. |
| NFC | Unicode canonical normalization form C. |
| NFKC | Unicode compatibility normalization form KC. |

## Scope

The required stages are layout admission, feature preparation, inference, reference parsing, marker detection and marker matching.
Reference parsing includes the required name and date stages.
Citation scope includes abstracts, body text, lists, notes, captions, annexes and appendices.
Bibliography labels are not citation callouts. Missing required region coverage must cause atomic failure.

The production input comes from Tracey's existing PDF owner.
The baseline may use PDFalto during local verification.
The fork must not depend on Tracey crates or personal data.

## Requirements and acceptance

Each requirement has one owner. Each acceptance check requires a recorded result.
The owner table follows the requirements. No runtime gate has passed at specification publication.

### GFR-001 — Fixed source identity

The module must use a fixed fork commit and verified model assets.
A branch name alone must not identify a release input.

1. Record the fork commit and model digests.
2. Repeat the build with those exact inputs.

### GFR-002 — Upstream attribution

The project must retain upstream licenses, notices, model identities and dataset citations.
New documentation must identify the work as a GROBID integration.

1. Compare upstream notices before and after the change.
2. Identify the original model and dataset authors in the release records.

### GFR-003 — One owner for each responsibility

The architecture must separate pipeline policy, model assets, inference, PDF input, application actions and evaluation.
The fork must have no dependency on Tracey application crates.

1. Examine the dependency graph and public exports.
2. Make sure that each domain rule has one owner.

### GFR-004 — Local production processing

Production extraction must run on the reader's device.
It must not require Docker, another production server or an external metadata service.

1. Prepare verified assets and process an admitted PDF with external network access blocked.
2. Make sure that extraction sends no document content.

### GFR-005 — Existing PDF owner

The production adapter must use Tracey's existing PDF engine and admitted document.
It must not introduce another PDF renderer or document lifetime.

1. Count PDF engine instances during reading and citation processing.
2. Make sure that both operations use the same admitted document identity.

### GFR-006 — Exact document input

Each layout input must identify the exact PDF digest and page count.
The host must supply layout from those admitted bytes.

1. Replace the PDF under the same paper identity.
2. Make sure that old layout and maps cannot supply current controls.

### GFR-007 — Complete layout provenance

Layout must retain source order, token boundaries, lines, blocks, fonts, styles and page frames required by the models.
Missing required information must produce an explicit failure.

1. Compare layout records with the baseline input.
2. Remove a required field and examine the failure.

### GFR-008 — Compatible text processing

Normalization and tokenization must reproduce the recorded upstream rules before unchanged models are accepted.
Unicode indexing and locale differences must be tested.

1. Compare token sequences for accents, ligatures, punctuation and supplementary characters.
2. Compare upstream and candidate transformations under different host locales.

### GFR-009 — Compatible feature inputs

Feature columns, templates, bins, boundary flags and lexical lookups must match the recorded model profile.
A changed schema requires a different profile and new verification.

1. Compare every feature column on the fixed feature corpus.
2. Supply a changed schema and make sure that inference cannot start.

### GFR-010 — Verified model assets

The asset owner must check model bytes against the recorded SHA-256 values.
Missing, damaged or unsupported assets must not cause model substitution.

1. Change one model byte and request processing.
2. Make sure that the result identifies the failed asset.

### GFR-011 — Complete resource closure

The accepted profile must identify every required lexicon, tokenizer resource and inference dependency.
The candidate model list alone must not establish readiness.

1. Remove each required resource in turn.
2. Make sure that no partial or substituted profile is accepted.

### GFR-012 — Explicit model family and flavor

The first candidate profile must include the six recorded default CRF models.
Deep Learning models and article flavors require separate profiles and evidence.
The verified dependency closure must identify any additional required upstream models before activation.

1. Supply an unsupported model family or flavor.
2. Make sure that the module reports incompatibility without fallback.

### GFR-013 — Reused inference

The implementation must first test established Wapiti inference with the unchanged models.
A new decoder requires evidence that reuse cannot satisfy the target contract.

1. Run upstream feature fixtures through both decoders.
2. Compare all labels, sequence boundaries and recorded tie cases.

### GFR-014 — No model training

The integration must not retrain, quantize, prune or replace trained parameters in the accepted reuse profile.
Such work requires a separately approved model activity.

1. Compare model digests before and after processing and packaging.
2. Make sure that no training task is part of the production build.

### GFR-015 — Upstream matching rules

Marker matching must preserve the required upstream numerical and author-year rules.
It must not substitute embeddings, a new fuzzy matcher or a language model.

1. Compare matching decisions with the pinned upstream matcher.
2. Examine repeated authors, year suffixes, punctuation and range limits.

### GFR-016 — Explicit uncertainty

Unresolved and ambiguous markers must stay distinct from linked markers.
The module must not invent catalogue identities or confidence values.

1. Process references without a verified match.
2. Make sure that no linked result or fabricated score is returned.

### GFR-017 — Citation groups

The mapping must retain all supported targets of grouped and ranged markers.
Shared locations must not produce overlapping independent controls.

1. Process grouped markers and expanded numerical ranges.
2. Examine retained targets, source text and the reader's choice control.

### GFR-018 — Verified geometry

Each marker box must identify its page frame and source location.
The document owner must transform boxes through the existing PDF display transform.

1. Check cropped, rotated and wrapped markers against reviewed PDF annotations.
2. Repeat the checks after zoom, resize and page reuse.

### GFR-019 — Stable occurrence provenance

New saved occurrence identities must include the complete map profile and exact document identity.
An old parser-order identity must not acquire a new meaning after reprocessing.

1. Reprocess one PDF with a changed profile.
2. Make sure that new occurrences cannot rebind old saved evidence.

### GFR-020 — Existing application contracts

The Tracey adapter must retain the existing citation-map, document receipt and canonical paper resolver owners.
The fork must return neutral reference data.

1. Examine dependencies and duplicate identity or cache implementations.
2. Make sure that only the existing resolver establishes canonical paper identity.

### GFR-021 — Complete cache identity

Cache identity must include document, runtime profile, PDF extractor, output adapter and map schema identities.
Any changed identity must invalidate active controls.

1. Change each identity separately.
2. Make sure that no old map is reused under the new identity.

### GFR-022 — Bounded processing

One validated policy must bound input, layout, assets, inference, output and concurrent jobs.
The module must report limits without returning a truncated complete map.
Host asset delivery must meet the existing hosting file limits.
Transport chunks must preserve the original model bytes and whole-file digest.

1. Exceed each bound independently.
2. Make sure that no partial result is presented as complete.

### GFR-023 — Cancellation and stale work

Cancellation, deadline expiry and obsolete generations must prevent publication.
The host must provide a bounded termination path when native inference cannot stop cooperatively.

1. Cancel during layout, features, inference and matching.
2. Make sure that reading remains available and obsolete results cannot change choices.

### GFR-024 — Atomic result publication

Only a complete validated mapping may be published.
Failure must preserve the current document, saved graph and previously verified cache entries.

1. Fail each stage before completion.
2. Compare document, graph and verified cache state with the initial state.

### GFR-025 — Explicit application actions

Extraction must not add papers or choose a trace.
Only an explicit reader action may request the existing guarded trace edit.

1. Complete extraction without selecting a marker.
2. Make sure that personal data and the selected paper stay unchanged.

### GFR-026 — Published evaluation inputs

Evaluation must reuse the fixed published GROBID PDF and gold XML pairs.
Generated GROBID TEI and PDFX files must not supply gold labels.

1. Verify every selected path, role and content identity.
2. Record the dataset revision, article notices and excluded files.

### GFR-027 — Reproducible baseline

Parity must use the same fixed source, models, flavor, explicit file list and consolidation settings.
Header, citation and funder consolidation must all be zero.

1. Run the baseline twice without external metadata access.
2. Compare selected files, configuration and outputs.

### GFR-028 — Independent quality scoring

Evaluation must report upstream agreement and gold accuracy separately for each corpus.
It must record missing links, range interiors, unsupported files and every denominator.

1. Score reference fields and labeled citation targets for both implementations.
2. Report every regression and unscored case before accepting the profile.

### GFR-029 — Separate hit-area checks

Publisher XML must not establish PDF hit-area correctness.
Geometry acceptance must use reviewed original-page annotations and actual reader interactions.

1. Check each annotated hit area on desktop and phone.
2. Make sure that activating the area selects its intended reference.

### GFR-030 — Measured resource use

Resource reports must record cold and warm time, peak memory, downloads, cancellation and reading responsiveness.
Results must identify the hardware, target, build and complete profile.

1. Run the fixed corpus on the supported browser and native targets.
2. Compare results with the recorded baseline and accepted device budgets.

### GFR-031 — Language decision

Rust is the production integration language. Python is the evaluation language.
A Python production alternative requires the same contracts and measured target-device results before replacement.

1. Record the runtime, packaging and resource differences for any alternative.
2. Make sure that no performance claim relies only on the language name.

### GFR-032 — Release gate

Tracey may pin an inactive candidate commit for development and documentation review.
Production activation must use a reviewed fork commit after all required runtime gates pass.
Documentation acceptance must not be reported as model or application acceptance.

1. Verify the fork branch, commit, profile and matching Tracey Git entry.
2. Record every unmet gate and keep production activation unavailable.

## Ownership and gates

Gate definitions and measurable comparisons are in [evaluation.json](../contracts/evaluation.json).
The API identifiers are defined in the [API specification](API_SPECIFICATION.md).

| Requirements | Owner | API or boundary | Required gates |
| --- | --- | --- | --- |
| GFR-001, GFR-010–GFR-014 | Assets and inference | F3–F7 | G0, G1 |
| GFR-002 | Release notices | Fork and package records | G0, G7 |
| GFR-003–GFR-004 | Architecture and host | F1, F5 | G0, G7 |
| GFR-005–GFR-009 | Layout and features | F1, F2 | G2, G4 |
| GFR-015–GFR-017 | Matching | F1 | G3, G4 |
| GFR-018, GFR-029 | Geometry and document adapter | F1, F2; parent transform | G5, G7 |
| GFR-019–GFR-021 | Profile and document adapter | F1, F4; parent map owner | G0, G7 |
| GFR-022–GFR-024 | Pipeline and host lifetime | F1, F2, F5, F6, F8 | G1, G6, G7 |
| GFR-025 | Tracey application actions | Existing guarded trace owner | G7 |
| GFR-026–GFR-028 | Evaluation | Python verification tools | G0, G4 |
| GFR-030–GFR-031 | Resource evaluation | Fixed target reports | G6 |
| GFR-032 | Release | Fork and parent Git records | G0–G7 |

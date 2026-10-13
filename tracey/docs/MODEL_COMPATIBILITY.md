# Model compatibility and reuse review

Version: 0.1. Date: 13 October 2026.
Baseline: `c71ebff51e4d2a8257885458daeedede82698d43`, GROBID `0.9.2-SNAPSHOT`.
Status: source review complete; inference and PDF compatibility unverified.

## Decision

Reuse the trained model bytes and the required upstream resources.
First test established Wapiti inference. Port the required compatibility layer around it.
Use the existing PDFium owner for production layout and rendering.
Keep the unchanged upstream Java implementation as a local verification reference.

Weights are independent data, but their meaning depends on the prepared features and decoder.
A successful model load does not establish compatible results.
The six recorded CRF files total 135,543,537 bytes.
This is file size. It does not establish decoded memory use or a complete dependency closure.
The [candidate profile](../contracts/model-profile.json) must remain unavailable until the required resources and target inference are verified.

## Reuse and adaptation decisions

| Component | Decision | Required evidence |
| --- | --- | --- |
| Trained CRF weights and templates | Retain exact bytes | File size and SHA-256 match |
| Required lexicons and dictionaries | Retain data and lookup meaning | Complete reached-resource list and lookup parity |
| Wapiti sequence inference | Reuse established source where compatible | Same-feature label parity and target build checks |
| Java normalization and feature code | Port required rules | Exact token and feature comparisons |
| Reference and marker assembly | Port required upstream behavior | Same-label assembly and target parity |
| Lucene-based matching behavior | Preserve the reached analyzer and term rules | Exact analyzer tokens and matching decisions |
| PDFalto | Retain for the local baseline | Fixed binary, flags and resources |
| Tracey PDFium | Retain for production | Verified layout adapter and independent geometry checks |
| Tracey receipt, map and cache | Retain existing owners | One output adapter and complete identity checks |
| Tracey catalogue and graph edits | Retain existing owners | Existing canonical resolution and guarded transaction checks |
| Python tooling | Use for verification | Fixed inputs, configuration and reports |

Do not rebuild the PDF parser, a general machine learning framework, model training, catalogue search or personal graph storage.
Do not replace a failed parity check with a new heuristic.
Compact derived resource indices require lookup equality tests and their own recorded build identity.

## Incompatibility triggers

### C01 — PDF layout changes

PDFalto and PDFium can produce different token boundaries, column order, fonts, clipped text, ligatures and source boxes.
Plain extracted text cannot substitute for the full required layout.
The baseline also controls PDFalto flags and language resources.

Compare ordered layout before feature generation.
Treat PDFium geometry improvements as explicit differences and check them against reviewed original pages.
Do not claim extractor parity from inference parity.

Sources: [DocumentSource.java](../../grobid-core/src/main/java/org/grobid/core/document/DocumentSource.java), lines 118–152;
[PDFALTOSaxHandler.java](../../grobid-core/src/main/java/org/grobid/core/sax/PDFALTOSaxHandler.java), lines 480–545.

### C02 — Contextual normalization

Upstream normalization uses ordered mappings for spaces, dashes, brackets and contextual characters.
Some transformations must occur before tokenization. Repeating them on subtokens can change the input.
Preserve original marker text separately from model feature text.

Use the same transformation order. Test middle dots, ligatures, combining characters and nonbreaking spaces.
Do not substitute general NFC or NFKC normalization without equality evidence.

Sources: [UnicodeUtil.java](../../grobid-core/src/main/java/org/grobid/core/utilities/UnicodeUtil.java), lines 193–264;
[CitationParser.java](../../grobid-core/src/main/java/org/grobid/core/engines/CitationParser.java), lines 93–114.

### C03 — Tokenization, character indexing and locale

The analyzer retains delimiter tokens and uses specific letter and digit splitting rules.
Java lengths, offsets and substring operations use UTF-16 units.
Rust byte counts or Unicode scalar counts can produce different prefixes and suffixes.
Case conversion also depends on the recorded Java locale and Unicode behavior.

Record the baseline JDK, locale and feature encoding.
Test supplementary characters, Turkish I, Arabic, CJK, non-ASCII digits and alphabetic-digit boundaries.
Preserve the observed encoding of split UTF-16 units in feature serialization.

Sources: [GrobidDefaultAnalyzer.java](../../grobid-core/src/main/java/org/grobid/core/analyzers/GrobidDefaultAnalyzer.java), lines 61–80, 106–119 and 223–247;
[TextUtilities.java](../../grobid-core/src/main/java/org/grobid/core/utilities/TextUtilities.java), lines 1268–1295;
[GrobidAnalyzer.java](../../grobid-core/src/main/java/org/grobid/core/analyzers/GrobidAnalyzer.java), lines 89–131.

### C04 — Feature columns and hidden dependencies

Model templates address exact feature columns and sequence positions.
Column order, defaults, spaces, blank boundaries and label-column handling must match.
Fulltext features include graphics proximity, known references, first-author information and superscript status.
Removing an output stage does not prove its input features unnecessary.

Compare serialized feature matrices exactly on identical layout inputs.
Keep required graphics and bibliography features even when their descriptive output is omitted.
Do not replace missing information with zero or false values to make a model run.

Sources: [FeaturesVectorFulltext.java](../../grobid-core/src/main/java/org/grobid/core/features/FeaturesVectorFulltext.java), lines 64–184;
[FullTextParser.java](../../grobid-core/src/main/java/org/grobid/core/engines/FullTextParser.java), lines 908–940 and 1308–1345.

### C05 — Numerical feature bins

The integer scaling overload uses float32 division and truncation.
The double overload uses float64. A universal float64 replacement can change boundary bins.
Position features also depend on upstream page and document denominators.

Compare serialized bin values exactly.
Test both overloads, edge positions, empty denominators and threshold neighbors.

Source: [FeatureFactory.java](../../grobid-core/src/main/java/org/grobid/core/features/FeatureFactory.java), lines 337–374.

### C06 — Lexical resources

Citation features use journal, publisher, conference, location, collaboration, identifier and URL matches.
Name and date features use additional dictionaries.
Removing resources changes feature meaning while model weights remain unchanged.
Lazy loading changes timing; it does not remove semantic dependencies.

Record the full reached-resource closure and exact lookup behavior.
Test token-position matches for any derived index before accepting it.

Sources: [CitationParser.java](../../grobid-core/src/main/java/org/grobid/core/engines/CitationParser.java), lines 163–184;
[Lexicon.java](../../grobid-core/src/main/java/org/grobid/core/lexicon/Lexicon.java), lines 113–138 and 258–290;
[FeatureFactory.java](../../grobid-core/src/main/java/org/grobid/core/features/FeatureFactory.java), lines 265–327.

### C07 — Stage order and consolidation

The upstream pipeline parses references before preparing body citation features.
Consolidation can change authors, dates and identifiers used by matching.
The full upstream process also handles headers, abstracts, captions and annexes.
The reduced scope must not silently lose citations in those regions.

Set header, citation and funder consolidation to zero in both parity runs.
Trace dependencies before omitting a stage. Record any unsupported region explicitly.
Do not use the unchanged evaluation defaults as a no-network baseline.

Sources: [FullTextParser.java](../../grobid-core/src/main/java/org/grobid/core/engines/FullTextParser.java), lines 158–281;
[EndToEndEvaluation.java](../../grobid-trainer/src/main/java/org/grobid/trainer/evaluation/EndToEndEvaluation.java), lines 112–126.

### C08 — Model family, flavor and fallback

CRF, Deep Learning and article-flavor models are different processing profiles.
The upstream configuration can inherit model settings or fall back when a model is unavailable.
The integration must fail explicitly instead of silently selecting another model.

The first profile must include the six recorded default CRF candidates.
Additional required models must be recorded before closure can pass.
Do not compare this profile with published Deep Learning and consolidation scores as if configurations were equal.

Source: [grobid.yaml](../../grobid-home/config/grobid.yaml), model settings and flavor inheritance.

### C09 — Native decoder and target support

The pinned fork supplies JNI wrappers, a Wapiti JAR and operating-system binaries.
It contains no tracked C/C++ source or WebAssembly decoder artifact.
Its arm64 Wapiti library targets macOS. The architecture name does not establish iOS compatibility.

The candidate [Wapiti source](https://github.com/kermitt2/Wapiti/tree/a9c25d2bcccd60f1a54a7019689bd5229e866f00) has not passed target or model parity checks.
Record its exact source, notices, compiler, flags and artifact digest before use.
Build for each required target. Do not load a macOS library in an iOS package.
Verify WebAssembly memory, instructions, file access and cancellation behavior.

Sources: [WapitiModel.java](../../grobid-core/src/main/java/org/grobid/core/jni/WapitiModel.java);
[native library guidance](../../doc/Recompiling-and-integrating-CRF-libraries.md);
the pinned `grobid-home/lib/mac_arm-64/libwapiti.dylib` platform record.

### C10 — Decoding and matching semantics

Wapiti source uses float64 decoding and ordered tie handling.
Different precision, label order, unknown-observation handling or sequence boundaries can change labels.
The current marker matcher uses Lucene term matching and author filters.
Its numerical range rule requires an endpoint difference below 20.
This path does not use `FuzzyQuery`.

Compare labels with identical features before comparing complete PDFs.
Compare analyzer tokens and matching decisions with identical reference and marker records.
Do not introduce embeddings, a new fuzzy matcher or a large language model as a compatibility fix.

Sources: [ReferenceMarkerMatcher.java](../../grobid-core/src/main/java/org/grobid/core/utilities/matching/ReferenceMarkerMatcher.java);
[LuceneIndexMatcher.java](../../grobid-core/src/main/java/org/grobid/core/utilities/matching/LuceneIndexMatcher.java);
[candidate decoder source](https://github.com/kermitt2/Wapiti/blob/a9c25d2bcccd60f1a54a7019689bd5229e866f00/src/decoder.c).

### C11 — Mapping identity and geometry

The same PDF can produce a different map after a profile change.
Parser-order keys such as `c0` must not rebind saved evidence.
Page width and height alone cannot establish alignment after crop or intrinsic rotation.

Include the complete map profile and PDF digest in new occurrence identities.
Retain MediaBox, CropBox, origin, user-unit scale and intrinsic rotation.
Use the existing PDF display transform and check actual hit areas independently.

Sources: [coordinate documentation](../../doc/Coordinates-in-PDF.md);
the [API identity and geometry contract](API_SPECIFICATION.md).

## Acceptance sequence

1. Verify exact assets, complete resources and target build provenance.
2. Compare decoder labels with identical upstream feature matrices.
3. Compare feature matrices with identical upstream layout records.
4. Compare assembly and matching with identical upstream labels and records.
5. Compare PDFium extraction against held-out gold data and the fixed baseline.
6. Verify original-page hit areas and target-device resource use.
7. Verify the existing reader, cache and guarded trace transaction.

A failed gate must identify its stage and smallest differing input.
An intentional improvement requires a new profile and separate quality evidence.
Model reuse is accepted only after the applicable gates pass.

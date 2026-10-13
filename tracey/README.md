# GROBID-fork: Tracey integration

This project adapts the GROBID citation pipeline for Tracey.
GROBID supplies the trained models and the reference implementation.
Retain the upstream [license](../LICENSE), notices and citations.
The integration does not claim authorship of the upstream models or datasets.

The specification version is 0.1. Runtime implementation is pending.
The upstream baseline is `c71ebff51e4d2a8257885458daeedede82698d43`.
It declares GROBID version `0.9.2-SNAPSHOT`.
The implementation branch is `feat/tracey-citation-mapping`.

Read these documents before implementation:

- [Activity specification](docs/SPECIFICATION.md)
- [Architecture specification](docs/ARCHITECTURE_SPECIFICATION.md)
- [API specification](docs/API_SPECIFICATION.md)
- [Model compatibility and reuse review](docs/MODEL_COMPATIBILITY.md)
- [Candidate model profile](contracts/model-profile.json)
- [Evaluation contract](contracts/evaluation.json)

These documents define required behavior. They do not establish runtime acceptance.
The model profile remains a candidate until its dependency and inference checks pass.

Keep new integration files under `tracey/`.
Keep upstream source, model files and documentation unchanged.
Use Rust for the shared production pipeline and Python for evaluation tools.
Use established native inference code where compatibility is verified.
Do not introduce Docker, a production server or another PDF rendering engine.

Tracey must pin the reviewed fork commit as a Git submodule.
A branch name identifies the work. The commit identifies a reproducible input.
Do not resolve a moving branch during a release build.

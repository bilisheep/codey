# Codey CLI Runtime for Python SDK

Platform-specific runtime package consumed by the published `codey-sdk`.

This package is staged during release so the SDK can pin an exact Codey CLI
version without checking platform binaries into the repo.

`codey-cli-bin` is intentionally wheel-only. Do not build or publish an
sdist for this package.

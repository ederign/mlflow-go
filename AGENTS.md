# MLflow Go SDK

This repository provides the Go SDK for MLflow. Public clients live under
`mlflow/`; shared HTTP, authentication, workspace, and artifact handling live
under `internal/`. Design decisions are recorded in `docs/adr/`.

## Development and validation

- Use the Go version and toolchain declared in `go.mod`.
- Run `make check` for lint, vet, and unit tests with the race detector.
- Check formatting with `gofmt -l .` and module changes with `go mod tidy`.
- `sample-app/` is a separate Go module; validate it separately when changed.
- Integration tests use the `integration` build tag and a running MLflow server.
  `make test/integration-ci` starts an isolated SQLite-backed server;
  `make test/integration-ci-midstream` exercises workspace support;
  `make test/integration-ci-postgres` requires Docker.
- Report commands actually run and any missing runtime dependencies. A skipped
  integration suite does not establish server compatibility.

## SDK contracts

- Preserve public API compatibility, context cancellation, typed API errors,
  pagination, and functional option semantics.
- Authentication and workspace changes need coverage for redirects,
  foreign-origin requests, explicit caller headers, and concurrent use.
- Artifact paths and URLs are trust boundaries. Preserve traversal checks and
  credential isolation between the tracking server and artifact storage.
- Keep upstream MLflow behavior distinct from midstream workspace behavior.
  Read the relevant ADR and existing tests before changing either contract.

## Generated code

`internal/gen/` contains generated protobuf code and fetched sources. Change
`tools/proto/` inputs and run `make gen` when regeneration is needed. CI checks
generation drift. OpenTelemetry messages use the official OTLP Go module;
preserve the protobuf registry collision regression test.

## Local configuration

Local environment files and scratch documents may exist in a developer's
checkout. Keep credentials and unrelated working files out of commits and
review output. Fullsend configuration lives under `.fullsend/`.

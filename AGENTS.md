# KServe

Serve ML models on Kubernetes: Go controllers/webhooks (`pkg/`, `cmd/`), CRDs and Helm under `config/` and `charts/`, Python SDK and model servers under `python/`.

## Build & test commands

**Offline unit tests (no cluster, no cloud credentials)** — primary agent loop:

**Go** — full suite with fmt, vet, manifest regeneration, envtest, and coverage (slowest; use before API/CRD changes):

```bash
make test
```

**Go** — faster iteration (envtest + tests only, no fmt/vet/manifests):

```bash
make envtest test-qpext
KUBEBUILDER_ASSETS="$(./bin/setup-envtest use $(go list -m -f '{{.Version}}' k8s.io/api | awk -F'[v.]' '{printf "1.%d", $3}') -p path)" \
  go test -timeout 30m $(go list ./pkg/...) ./cmd/...
```

Single package or subtree: replace `$(go list ./pkg/...) ./cmd/...` with e.g. `./pkg/controller/v1beta1/...`.

**Python** (aligned with CI: `python/kserve` + `python/storage`):

```bash
cd python/kserve && make install_dependencies && make dev_install    # once
cd python && source kserve/.venv/bin/activate && pytest ./kserve ./storage
```

One-shot with `uv` from `python/kserve` (same extras as `dev_install`: `test`, `ray`, `llm`):

```bash
cd python/kserve && uv sync --group test --extra ray --extra llm && uv pip install ../storage --no-cache && uv run pytest . ../storage
```

**Platforms:** **`uv sync`** with **`llm`** pulls **vLLM**, which has **no macOS wheels** — use **Linux** / **WSL2** / CI for that one-shot line, or use **`pytest`** after **`dev_install`** on macOS if your environment resolves deps. **Go** tests need a Go toolchain matching **`go.mod`** (and envtest assets via **`make envtest`**).

**Integration / cluster / E2E** — require Kubernetes (see `test/` and docs), not the commands above.

## Lint & format (fast single-path)

Repo-root Go helpers live under `bin/` after tool installs (via `make` targets).

```bash
make go-lint                          # all Go packages (golangci-lint)
bin/golangci-lint run --fix path/to/file.go   # single file or dir

make vet                              # go vet all packages
go vet ./pkg/controller/v1beta1/...   # single subtree

make py-lint                          # Ruff on Python trees (uses bin/.venv/ruff)
bin/ruff check path/to/file.py --fix --config ruff.toml

make py-fmt                           # Ruff format Python/docs/test trees
bin/ruff format path/to/file.py --config ruff.toml

make fmt                              # go fmt ./pkg/... ./cmd/...
```

There is no repo-wide Python typecheck in CI for all modules; `python/kserve` exposes `make type_check` (mypy for that package).

## Key conventions

- Changing CRDs or Go APIs: run **`make manifests`** (and related **`make generate`** when types/codegen change); generated YAML must stay in sync for tests under `test/crds/`.
- Large portions of **`python/kserve/kserve/models`** (and related generated REST/client pieces) are **generated** — regenerate via project scripts (`hack/`, `make generate`), don’t hand-edit like normal library code.
- **Serving reconcilers and webhooks:** `pkg/controller/` by API version; cluster-facing binaries **`cmd/manager`**, **`cmd/agent`**, **`cmd/router`**.
- **`make check`** (CI) runs **`make precommit`** and fails if the git tree is dirty — run **`make precommit`** locally before pushing when changing Go/Python/chart assets.

## Architecture (routing)

| Area | Location |
|------|----------|
| InferenceService / controllers | `pkg/controller/v1beta1/`, `pkg/apis/serving/` |
| InferenceGraph, TM, local model | `pkg/controller/v1alpha1/` |
| LLMISVC / Gateway integration | `pkg/controller/v1alpha2/`, `config/crd/full/llmisvc/` |
| Envtest CR fixtures | `test/crds/` |
| Python SDK source | `python/kserve/kserve/` |
| Storage init / URI handling | `python/storage/` |
| Sample model servers | `python/sklearnserver/`, `python/xgbserver/`, etc. |

## PR / CI conventions

- **`make check`** must pass on PRs that touch non-markdown code (see `.github/workflows/precommit-check.yml`).
- For offline Go/Python changes, **`make test`** and **`pytest ./kserve ./storage`** (from `python/` with the kserve venv) are the main local gates; optional servers are tested in **`.github/workflows/python-test.yml`**.
- Contributor policy and DCO/sign-off expectations: [KServe community — How to contribute](https://github.com/kserve/community#how-can-i-help-).

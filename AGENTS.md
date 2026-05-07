# KServe

Serve ML models on Kubernetes: Go controllers/webhooks (`pkg/`, `cmd/`), CRDs and Helm under `config/` and `charts/`, Python SDK and model servers under `python/`.

## Build & test commands

**Offline unit tests (no cluster, no cloud credentials)** — primary agent loop:

```bash
make test-unit                 # Go controller tests + Python kserve & storage
make test-unit-go            # Go only (envtest + local apiserver; downloads kube test binaries once)
make test-unit-python        # Python only (uv sync + pytest under python/kserve)
```

**Platforms:** `make test-unit-python` installs the same extras as CI (`ray`, `llm` / vLLM). **vLLM does not publish wheels for macOS**, so `uv sync` may fail there — use **Linux** (or WSL2 / your CI image) for full Python unit tests, or run **`make test-unit-go`** locally and rely on **`python-test`** workflow for Python. Go tests only need a Go toolchain compatible with `go.mod` and envtest assets.

**Scope to one area:**

```bash
make test-unit-go GO_TEST_PKGS="./pkg/controller/v1beta1/..."
make test-unit-python PACKAGE="test/test_infer_type.py ../storage"
```

From `python/` with `kserve`’s venv active (`cd python/kserve && make dev_install`):  
`pytest ./kserve ./storage` — same coverage as the default `PACKAGE`; narrow with a file under `./kserve/test/` or `-k expression`.

**Full Go CI-like run** (fmt, vet, regenerates manifests — slow; run before commits touching APIs or CRDs):

```bash
make test                    # includes manifests generation + envtest + coverage
```

**Integration / cluster / E2E** — not covered by `make test-unit`; require Kubernetes (examples under `test/` and docs).

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
- **`make test-unit`** should pass for offline changes affecting Go or core Python SDK/storage.
- **`pytest`** matrices for optional servers live in `.github/workflows/python-test.yml` when under `python/**`.
- Contributor policy and DCO/sign-off expectations: [KServe community — How to contribute](https://github.com/kserve/community#how-can-i-help-).

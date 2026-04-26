# Release Notes — 0.39.0

This release is a major rework of the `--ai-annotate` backend system and adds first-class WSL support for the `--show` flag.

---

## Highlights

- **`--ai-annotate bedrock` now uses `boto3` directly.** No more API Gateway proxy stack to deploy. Authentication uses the standard AWS credential chain (env vars, `~/.aws/credentials`, IAM role, SSO).
- **New `--ai-annotate restapi` backend.** Point TerraVision at any OpenAI-compatible `/v1/chat/completions` endpoint — OpenAI, Anthropic via LiteLLM, vLLM, LM Studio, OpenRouter, custom proxies, etc.
- **WSL users no longer need a system-wide `xdg-open` shim** to use `--show`. TerraVision auto-detects WSL and routes through `wslview` from the `wslu` package.

---

## ⚠️ Breaking changes

### `--ai-annotate bedrock` requires AWS credentials, not an API Gateway URL

The previous `bedrock` backend POSTed to a hardcoded API Gateway URL (`yirz70b5mc.execute-api.us-east-1.amazonaws.com`) backed by a Lambda + Bedrock proxy that users were expected to deploy from `ai-backend-terraform/`. That entire indirection has been removed.

**What changed**

- `BEDROCK_API_ENDPOINT` constant **removed** from `modules/config/cloud_config_aws.py`, `cloud_config_azure.py`, and `cloud_config_gcp.py`.
- `_stream_bedrock_text()` now calls `bedrock-runtime.converse_stream` via `boto3` directly. Streaming uses the AWS EventStream protocol (parsed by botocore) and emits `contentBlockDelta` events for incremental output.
- `check_bedrock_endpoint(url)` preflight replaced with `check_bedrock_credentials()`, which calls `sts:GetCallerIdentity` so preflight needs no Bedrock IAM permissions.

**What you need to do**

1. Make sure your AWS credentials are configured (env vars, `~/.aws/credentials`, IAM role, or SSO — anything that works for `aws sts get-caller-identity` works here).
2. Optional: override defaults via env vars.
   - `TV_BEDROCK_REGION` — defaults to `us-east-1`
   - `TV_BEDROCK_MODEL_ID` — defaults to `us.anthropic.claude-haiku-4-5-20251001-v1:0`. Any Converse-compatible model id works (Claude family, Nova, Llama, Mistral); cross-region inference profile ids are accepted as-is.
3. You can delete any `ai-backend-terraform/` Lambda + API Gateway stack you previously deployed — it is no longer needed.

```bash
# Before (no longer works)
# (relied on a hardcoded API Gateway URL)

# After
export TV_BEDROCK_REGION=eu-west-1                                            # optional
export TV_BEDROCK_MODEL_ID=us.anthropic.claude-sonnet-4-5-20250929-v1:0       # optional
terravision draw --source ./infra --ai-annotate bedrock
```

---

## ✨ New features

### `--ai-annotate restapi` — generic OpenAI-compatible endpoint

A third backend choice that POSTs an OpenAI-style chat-completions request and parses the streamed SSE response. Works with anything that speaks the OpenAI schema.

**Configuration** (all three required; preflight fails fast if missing):

| Variable | Description |
|---|---|
| `TV_RESTAPI_URL` | Full URL including the `/v1/chat/completions` path |
| `TV_RESTAPI_KEY` | Bearer token sent as `Authorization: Bearer <key>` |
| `TV_RESTAPI_MODEL` | Model id passed through verbatim in the request payload |

**Examples**

```bash
# OpenAI direct
export TV_RESTAPI_URL=https://api.openai.com/v1/chat/completions
export TV_RESTAPI_KEY=sk-...
export TV_RESTAPI_MODEL=gpt-4o-mini
terravision draw --source ./infra --ai-annotate restapi

# Anthropic via LiteLLM proxy
export TV_RESTAPI_URL=https://your-litellm-proxy.example/v1/chat/completions
export TV_RESTAPI_KEY=sk-litellm-...
export TV_RESTAPI_MODEL=claude-haiku-4-5
terravision draw --source ./infra --ai-annotate restapi

# Local vLLM / LM Studio
export TV_RESTAPI_URL=http://localhost:8000/v1/chat/completions
export TV_RESTAPI_KEY=not-needed
export TV_RESTAPI_MODEL=meta-llama/Llama-3.1-8B-Instruct
terravision draw --source ./infra --ai-annotate restapi
```

### WSL detection for `--show`

TerraVision now auto-detects WSL at runtime and routes diagram opens through `wslview` (from the `wslu` package) instead of the broken `xdg-open` lookup chain that ships in WSL images by default. This affects all three opener call sites equally:

- Graphviz/Canvas (PNG, SVG, PDF, etc.) — `view=False` is forced on WSL and `wsl_open()` is invoked after render
- `--format drawio` — `click.launch()` swapped for `wsl_open()` on WSL
- `terravision visualise` (HTML output) — `webbrowser.open()` swapped for `wsl_open()` on WSL

On non-WSL platforms the existing opener for each path is preserved verbatim — no behavior change on macOS, Linux, or Windows.

**Setup for WSL users**

```bash
sudo apt install wslu
```

If `wslu` is missing, TerraVision prints a one-line install hint and continues — the diagram is always generated correctly regardless. `--show` only controls auto-opening.

---

## 📦 Dependencies

- **Added**: `boto3>=1.35.0` (and transitives: `botocore`, `jmespath`, `urllib3`, `s3transfer`). Lazy-imported inside the bedrock backend so users on `ollama` / `restapi` don't pay the import cost at startup.
- **WSL-only**: `wslu` (apt package) is recommended for `--show`. Optional — diagrams generate without it.

---

## 📚 Documentation

Comprehensive doc updates to reflect the new backend model:

- [README.md](README.md) — three-backend example block + CLI table updated.
- [docs/usage-guide.md](docs/usage-guide.md) — new "Choosing a backend" comparison table; new "Configuration" subsection covering all five env vars with worked examples for OpenAI / LiteLLM / vLLM.
- [docs/faq.md](docs/faq.md) — "Where does the data go?" paragraph added; backend list extended.
- [docs/installation.md](docs/installation.md) — `wslu` listed under System Requirements; new "WSL (Windows Subsystem for Linux)" subsection under Step 1.
- [docs/troubleshooting.md](docs/troubleshooting.md) — new entry: "`--show` doesn't open the diagram on WSL".
- [docs/cicd-integration.md](docs/cicd-integration.md) — CI examples for all three backends, including a GitHub Actions OpenAI/restapi sample.
- [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md) — testing setup updated for all three backends.
- [docs/CLAUDE.md](docs/CLAUDE.md) — AI Annotation Pipeline section rewritten; old API Gateway / `ai-backend-terraform/` description replaced with the boto3 + Converse story.

---

## 🧪 Tests

- [tests/test_ai_annotations.py](tests/test_ai_annotations.py) — bedrock unreachable test rewritten to raise `botocore.exceptions.ClientError` (the realistic exception type for the new boto3 path). Three new tests: restapi happy path, restapi with missing env vars, and unknown-backend rejection.

---

## 🔧 Internal / refactoring

- New module constants and helpers in [modules/llm.py](modules/llm.py): `_bedrock_region()`, `_bedrock_model_id()`, `_restapi_settings()`.
- Three preflight checks: `check_ollama_server()`, `check_bedrock_credentials()`, `check_restapi_endpoint()`.
- Three streamers: `_stream_ollama_text()`, `_stream_bedrock_text()` (boto3 Converse), `_stream_restapi_text()` (OpenAI SSE).
- New WSL helpers in [modules/helpers.py](modules/helpers.py): `is_wsl()` (cached after first probe) and `wsl_open()` (WSL-only, callers must gate with `is_wsl()`).
- [resource_classes/__init__.py](resource_classes/__init__.py) `Canvas.render()` — adds WSL fork that forces `view=False` to graphviz and routes through `wsl_open()` instead.

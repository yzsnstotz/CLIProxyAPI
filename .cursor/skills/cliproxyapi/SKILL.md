---
name: cliproxyapi
description: Configures CLIProxyAPI for a specified scenario and verifies the API works end-to-end. Use when the user wants to set up the proxy API for a scenario (e.g. OpenAI-compatible client, Amp CLI, local dev), wire config, start the service if needed, and verify with real requests until successful (调通).
---

# CLIProxyAPI Scenario Setup

When invoked, configure the CLIProxyAPI service for the **specified scenario** and **verify the API works** (调通) by running checks until they succeed.

## Scenarios

| Scenario | Purpose | Verification |
|----------|---------|--------------|
| `openai` | OpenAI-compatible clients (base URL `http://localhost:8317/v1`) | GET /v1/models with api-key returns 200 |
| `local` | Local dev / minimal | Server responds on port 8317, GET /v1/models or GET / |
| `amp` | Amp CLI / IDE | Amp provider routes and management API respond |

Default scenario if not specified: **openai**.

## Workflow

1. **Determine scenario**  
   From user message: openai, local, amp, or other. Default to `openai`.

2. **Ensure config file exists**  
   - Config path: workspace `config.yaml`. If missing, copy from `config.example.yaml`.  
   - Resolve auth-dir (e.g. `~/.cli-proxy-api`) per project.

3. **Apply scenario-specific config**  
   - **openai**: Ensure `api-keys` has at least one key; client will use `Authorization: Bearer <key>`.  
   - **local**: Ensure `port: 8317` (default); optional api-keys.  
   - **amp**: Ensure `ampcode` section if Amp is needed; set `remote-management.secret-key` if using management API.  
   - **OAuth providers (e.g. antigravity)**: Ensure there is an auth entry for that provider (e.g. OAuth with provider `antigravity`). The client must then use **model IDs** from `GET /v1/models` (e.g. `gemini-2.0-flash`), not the provider name or `antigravity-cli`.  
   Edit `config.yaml` only as needed for the chosen scenario.

4. **Ensure server is running**  
   - If nothing is listening on the configured port (default 8317), start the server:  
     `go run ./cmd/server -config <config_path>` from repo root (or use the built binary).  
   - Use a short wait (e.g. 2–3s) then check again if startup is slow.

5. **Verify until success**  
   - **openai**:  
     `curl -sS -o /dev/null -w "%{http_code}" -H "Authorization: Bearer <api-key>" http://127.0.0.1:8317/v1/models`  
     Expect 200. Use the first key from `config.api-keys` if the user did not specify one.  
   - **local**:  
     `curl -sS -o /dev/null -w "%{http_code}" http://127.0.0.1:8317/`  
     Expect 200.  
   - If the check fails: read response body/headers, fix config or startup (e.g. add api-keys, fix port, restart server), then retry.  
   - Repeat until the verification request returns success (200).  
   - **Optional (openai)**: After 200, fetch the full model list and cache it:  
     `curl -sS -H "Authorization: Bearer <api-key>" http://127.0.0.1:8317/v1/models` → parse `data[].id` (and optionally `owned_by`), write to `.cursor/cliproxyapi-models.json` for later use when choosing a `model` for chat/completions or when the user asks for a specific provider (e.g. antigravity).

6. **Report**  
   - State the scenario, config path, and that the API is verified (调通).  
   - Give the client-ready base URL and auth hint, e.g.:  
     Base URL: `http://127.0.0.1:8317/v1`, Auth: `Authorization: Bearer <api-key>`.  
   - Remind the user: use a **model ID** from `GET /v1/models` (or from `.cursor/cliproxyapi-models.json` if cached) as the `model` parameter—do not use provider names like `antigravity-cli`.

## Important details

- **Port**: Default 8317 (from `config.port`).  
- **API key**: From `config.api-keys[0]` unless the user provides another.  
- **Management API**: Only enabled when `remote-management.secret-key` is set.  
- **Auth dir**: `config.auth-dir` (e.g. `~/.cli-proxy-api`) for OAuth tokens; ensure it is writable when using OAuth providers.

## Provider vs model (avoiding 502 "unknown provider for model xxx")

The proxy routes by **model ID** only. The `model` parameter in requests **must** be one of the **model IDs** returned by `GET /v1/models`. It must **not** be:

- A provider or channel name (e.g. `antigravity`, `gemini-cli`).
- A composite like `antigravity-cli` or `provider-model`.

Using a non-registered string as `model` causes **502** and the error message **"unknown provider for model &lt;that string&gt;"**.

- **Config side**: For OAuth providers (e.g. antigravity), ensure `config.yaml` has the corresponding auth (e.g. OAuth entry with provider `antigravity`). The proxy then registers that provider’s models (e.g. from upstream antigravity API) into the global model list.
- **Client side**: Use a **model ID** from `GET /v1/models` (e.g. `gemini-2.0-flash`, `gemini-2.5-flash`) as the `model` in `POST /v1/chat/completions`. For antigravity-backed models, the IDs are the upstream model names (e.g. `gemini-2.0-flash`), not `antigravity-cli`.

## Fetch and cache model list (recommended)

After `GET /v1/models` returns 200:

1. **Fetch the full list**:  
   `curl -sS -H "Authorization: Bearer <api-key>" http://127.0.0.1:8317/v1/models`  
   Parse the JSON and extract from `data[]`: `id`, and optionally `owned_by` / root-level metadata.

2. **Cache for quick reference**:  
   Write a minimal cache file, e.g. workspace `.cursor/cliproxyapi-models.json`, with an array of objects like `{"id":"<model-id>","owned_by":"<optional>"}`. This allows the skill and the user to choose a valid `model` without calling the API every time.

3. **When the user wants a specific provider (e.g. antigravity)**:  
   Prefer model IDs whose `owned_by` or type indicates that provider (e.g. antigravity). If the cache is missing or stale, call `GET /v1/models` again and refresh the cache.

## Optional: stronger verification (openai)

After GET /v1/models succeeds (and ideally after caching the model list):

1. **Pick a model**: Use the first available model ID from the cache (`.cursor/cliproxyapi-models.json`), or from a fresh `GET /v1/models` response. If the user asked for antigravity, prefer a model whose `owned_by`/type suggests antigravity. Do **not** use a placeholder like `gpt-4` or `antigravity-cli` unless it appears in the model list.

2. **Call chat completions**:

```bash
curl -sS -X POST http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"<model-id-from-list>","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

If this returns 200 and a JSON body, the scenario is fully 调通. If it returns 502 "unknown provider for model ...", the `model` value is not in the registry—use a model ID from GET /v1/models (or the cache). If it returns other 4xx/5xx, fix credential or config and retry.

## Reference

- Config template and all keys: [config.example.yaml](../../config.example.yaml) in repo root.  
- Management API: see [reference.md](reference.md) for endpoints and api-call usage.

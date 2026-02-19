# CLIProxyAPI Skill Reference

## Management API

- Base path: `/v0/management`
- Auth: `Authorization: Bearer <secret-key>` or `X-Management-Key: <secret-key>`
- Only available when `remote-management.secret-key` is set in config.

### Useful endpoints

- `GET /v0/management/config` – current config (when permitted)
- `GET /v0/management/auth-files` – list auth credentials (auth_index for api-call)
- `POST /v0/management/api-call` – proxy HTTP request with optional `auth_index`, `method`, `url`, `header`, `data`; supports `$TOKEN$` in headers

### Example: api-call (test upstream with credential)

```bash
curl -sS -X POST "http://127.0.0.1:8317/v0/management/api-call" \
  -H "Authorization: Bearer <MANAGEMENT_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"method":"GET","url":"https://api.openai.com/v1/models","header":{"Authorization":"Bearer $TOKEN$"}}'
```

## Public API (client-facing)

- `GET /` – root, returns 200 and endpoint list
- `GET /v1/models` – list models (OpenAI-compatible); requires api-key when auth is enabled. Response includes `data[]` with `id`, `owned_by`, etc. Use these **model IDs** as the `model` parameter in chat/completions—do not use provider names (e.g. `antigravity-cli`).
- `POST /v1/chat/completions` – chat completions (OpenAI-compatible). The `model` field must be one of the IDs from `GET /v1/models`; otherwise the proxy returns 502 "unknown provider for model ...".

Default port: 8317.

## Model list cache (skill use)

The skill may cache the result of `GET /v1/models` to workspace `.cursor/cliproxyapi-models.json` (array of `{"id":"<model-id>","owned_by":"<optional>"}`) so that a valid `model` can be chosen without calling the API every time. For antigravity, use model IDs from this list whose `owned_by`/type indicates antigravity (e.g. upstream names like `gemini-2.0-flash`), not the string `antigravity-cli`.

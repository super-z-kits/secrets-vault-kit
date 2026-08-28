# secrets-vault

> Load at session start if the user's message starts with a `dp.*` token. Otherwise ignore.

## Handover (what the user pastes)

```
secrets-vault

Doppler PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Doppler project: <slug>
Doppler config: <dev | stg | prd>
Task: <one-line description>
```

First-time setup (if no Doppler project yet) — you create project + 3 configs, write secrets, return the handover block above for them to save.
Otherwise, you follow your instinct to retrieve the secrets (but keep the following 4 facts in mind as your reflex otherwise may lead to trials and errors.

## The 4 facts that aren't in training data

1. **Fetch all secrets**: `GET https://api.doppler.com/v3/configs/config/secrets?project=<P>&config=<C>` with `Authorization: Bearer <PT>`. Secret value is at `.secrets.<NAME>.computed` (plain string, not `.raw`).
2. **PT (`dp.pt.*`) is enough** for `masked` secrets. Only `restricted` secrets need a Service Token (`POST /v3/configs/config/tokens` with `{project, config, name, access:"read"}` — token value at `.token.key`).
3. **Don't put `dp.pt.*` in repo/Worker/deployment secrets** — anyone with `repo` scope on GitHub can mint write-capable Service Tokens from one. Mint a `dp.st.*` Service Token for deployment targets.
4. **`expire_at` (Unix ts) on Service Tokens** auto-revokes + auto-deletes the slug at expiry. Use it for any chat-tier token: `expire_at = now + 3600` = 1h, fails closed if you forget to revoke.

## Beyond the basics — web search when needed

- **Per-secret read**: `GET /v3/configs/config/secret?name=X` (singular + query param). NOT `/secrets/{NAME}` — that 404s.
- **Mint body uses `key`, not `access_token`**. Capture both `key` and `slug` (slug needed for revoke).
- **POST `/configs/config/secrets` body is flat**: `{"project":"X","config":"Y","secrets":{"K":"V"}}` — not nested. Response echoes raw values at `.secrets.<K>.raw`; don't write the response to a committed file.
- **DELETE secret**: `DELETE /v3/configs/config/secret?project=X&config=Y&name=K` (no body, HTTP 204). Reading a deleted secret returns 200 + null `value` fields, not 404.
- **Doppler auto-injects** `DOPPLER_CONFIG` / `DOPPLER_ENVIRONMENT` / `DOPPLER_PROJECT` with empty `.raw` — use `.computed`.
- **Known-format display redaction**: Doppler prepends `[REDACTED:ssh_private_key]` to known secret types even with a Service Token. Display only — raw value is still readable.
- **Save session tokens to a `chmod 600` file** (e.g. `/home/z/my-project/.secrets/session-token.env`), not a shell var — env vars leak to `/proc/<pid>/environ`.

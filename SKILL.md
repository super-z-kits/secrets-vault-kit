# secrets-vault

> Load at session start if the user pastes a `dp.*` token. Otherwise ignore.

## Handover (what the user pastes)

```
Doppler PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
project: example-project
config: dev
<task description>
```

First-time setup (if no Doppler project yet) — create project + 3 configs (dev/stg/prd), write the secrets, return the handover block above for the user to save.

## The 5 facts that aren't in training data

1. **Fetch all secrets**: `GET https://api.doppler.com/v3/configs/config/secrets?project=<P>&config=<C>` with `Authorization: Bearer <PT>`. Secret value is at `.secrets.<NAME>.computed` (plain string, not `.raw`). Response also includes auto-injected `DOPPLER_CONFIG` / `DOPPLER_ENVIRONMENT` / `DOPPLER_PROJECT` with empty `.raw` — use `.computed` for those too.
2. **Use the PT directly for the chat session.** It's already in chat (burned). Don't mint a Service Token as a "least-privilege" reflex — the agent sees the PT anyway, so the ST doesn't reduce blast radius. Minting ceremony is theater for the chat session.
3. **Only mint a Service Token when putting a credential in a persistent deployment target** (GitHub repo secret, Cloudflare Worker secret, Vercel/Netlify env var). Reason: anyone with `repo` scope on GitHub can read a stored secret; if it's a `dp.pt.*` they can mint write-capable Service Tokens from it. An ST (`dp.st.*`) is scoped to one (project, config), read-only by default, TTL-bounded via `expire_at` (Unix ts; auto-revokes + auto-deletes the slug at expiry).
4. **Don't write secret values to git-tracked files.** POST `/configs/config/secrets` echoes raw values back at `.secrets.<K>.raw` in the response body — don't write the response to a file you'll commit. Once a secret is in the LLM's context (fetched from Doppler or echoed in a response), it's in the provider's request logs regardless of what the agent does with stdout — don't waste cycles on "don't echo to terminal" theater.
5. **Super Z bash tool redacts known token prefixes (`ghp_*`, `dp.*`, `cfat_*`, `sbp_*`) in display output** — the response contains the real value, you just can't SEE it. Don't waste calls chasing "Doppler is restricting the secret." Capture to a shell var and pipe straight to the next call: `GH_PAT=$(curl -s -H "Authorization: Bearer $PT" "https://api.doppler.com/v3/configs/config/secrets?project=$P&config=$C" | jq -r '.secrets.GH_PAT.computed')` then `curl -H "Authorization: Bearer $GH_PAT" ...`. Verify by length: `echo "${#GH_PAT}"`.

## Situational specifics

**Reading one secret by name** (instead of fetching all): `GET /v3/configs/config/secret?name=X` (singular + query param). NOT `/secrets/{NAME}` — that 404s. Value at `.value.computed`.

**Minting a Service Token** (only for deployment targets — see fact #3): `POST /v3/configs/config/tokens` with body `{"project":"X","config":"Y","name":"agent-<ts>","access":"read","expire_at":<unix_ts>}`. Response: `{"token":{"key":"dp.st.*","slug":"<UUID>","expires_at":"<ISO>",...}}`. Capture BOTH `key` (use) and `slug` (revoke). Save the key to a `chmod 600` file (e.g. `/home/z/my-project/.secrets/deploy-token.env`), not a shell var — env vars leak to `/proc/<pid>/environ`.

**Writing a secret** (only with the user's explicit ask): `POST /v3/configs/config/secrets` with body `{"project":"X","config":"Y","secrets":{"KEY":"value"}}` (flat string values, not nested — nested returns 400 "must be a string").

**Deleting a secret**: `DELETE /v3/configs/config/secret?project=X&config=Y&name=K` (no body, HTTP 204). Reading a deleted secret returns 200 + null `value` fields, not 404.

**Revoking a Service Token**: `DELETE /v3/configs/config/tokens/token` with body `{"project":"X","config":"Y","slug":"<UUID>"}`. Returns 200 + `{"success":true}` (not 204).

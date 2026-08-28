# secrets-vault-skill

> Read this at session start if the user pastes a `dp.*` token. Otherwise ignore.
>
> **Pair with `z-container-kit`** (https://github.com/super-z-kits/z-container-kit) — load both at session start. z-container governs persistence (git IS the disk here — `.env` is committed by design, see z-container law 9); this kit governs the Doppler vault. The two compose: the Doppler PT lives in the committed `.env`; secrets you fetch FROM Doppler for runtime use are not committed (they live in Doppler, your repo only references them by name).

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
3. **Don't put `dp.pt.*` in persistent deployment targets** (GitHub repo secrets, Cloudflare Worker secrets, Vercel/Netlify env vars) — anyone with `repo` scope on GitHub can read a stored secret, and from a `dp.pt.*` they can mint write-capable Service Tokens. Mint a `dp.st.*` Service Token instead (scoped to one project+config, read-only by default, TTL-bounded via `expire_at` Unix ts which auto-revokes + auto-deletes the slug at expiry). See `SKILL-DEPLOY.md` for the deployment flow.
4. **Store the PT in `/home/user_skills/zk-doppler.env` (mode 0600)** — NOT in the project's `.env` and NOT in a shell var. Reasons: (a) `/home/user_skills/` is OUTSIDE the project repo, so git/trufflehog/gitleaks/GitHub push protection can't see it — code reviewers won't flag it and GitHub won't auto-revoke a `ghp_*` if it leaks there; (b) it survives recycles + force-kills (PolarFS) and probably survives into a new chat (per-user namespace, see z-container persistence map); (c) mode 0600 means other processes on the box can't read it; (d) matches the z-container `zk-remote.url` pattern — same place, same posture. Canonical file contents:
   ```
   DOPPLER_PT=dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   DOPPLER_PROJECT=example-project
   DOPPLER_CONFIG=dev
   ```
   Source at the start of each Bash call that needs it: `set -a; source /home/user_skills/zk-doppler.env; set +a` (each Bash toolcall is a fresh subshell — don't assume vars persist across calls; safest pattern is to source + fetch + use inside a single call, as fact #5 shows). Secrets you FETCH from Doppler for runtime use (GH_PAT, STRIPE_KEY, etc.) go in shell vars only — they're already in Doppler permanently, your repo references them by name, never writes their values to a tracked file. POST `/configs/config/secrets` echoes raw values back at `.secrets.<K>.raw` in the response body — don't write that response to a file you'll commit.
5. **Super Z bash tool redacts known token prefixes (`ghp_*`, `dp.*`, `cfat_*`, `sbp_*`) in display output** — the response contains the real value, you just can't SEE it. Don't waste calls chasing "Doppler is restricting the secret." Capture to a shell var and pipe straight to the next call: `GH_PAT=$(curl -s -H "Authorization: Bearer $PT" "https://api.doppler.com/v3/configs/config/secrets?project=$P&config=$C" | jq -r '.secrets.GH_PAT.computed')` then `curl -H "Authorization: Bearer $GH_PAT" ...`. Verify by length: `echo "${#GH_PAT}"`.

🚨 **Don't revoke the user's Personal Token.** It's their master key — they rotate it themselves via the Doppler dashboard after the session. If you accidentally revoke it (e.g. via `DELETE /v3/workplace/personal_tokens/...`), the user loses access to their vault and you've made a mess.

## Situational specifics (rare ops)

**Reading one secret by name** (instead of fetching all): `GET /v3/configs/config/secret?name=X` (singular + query param). NOT `/secrets/{NAME}` — that 404s. Value at `.value.computed`.

**Writing a secret** (only with the user's explicit ask): `POST /v3/configs/config/secrets` with body `{"project":"X","config":"Y","secrets":{"KEY":"value"}}` (flat string values, not nested — nested returns 400 "must be a string").

**Deleting a secret**: `DELETE /v3/configs/config/secret?project=X&config=Y&name=K` (no body, HTTP 204). Reading a deleted secret returns 200 + null `value` fields, not 404.

## When you need to read SKILL-DEPLOY.md

If the task involves deploying to a persistent target (GitHub Actions workflow, Cloudflare Worker, Supabase Edge Function, Vercel/Netlify) and wiring secrets into it — read [`SKILL-DEPLOY.md`](./SKILL-DEPLOY.md) in this repo. It covers:
- Minting a `dp.st.*` Service Token for the deployment target (the one legitimate ST use case)
- GH Actions pull-at-runtime pattern (`doppler run` in the workflow)
- Cloudflare Workers DIY pull (Doppler has no auto-sync to Workers — only Pages)
- Supabase one-click auto-sync OR manual Management API
- Rotation mechanics per platform (chain-reaction resolved via seed/worker split)

Don't read SKILL-DEPLOY.md for: reading secrets to use in chat, fetching a token to call an API directly, one-shot tasks, or local dev work.

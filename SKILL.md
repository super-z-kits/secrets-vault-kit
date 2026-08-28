# secrets-vault-skill

> Read this at session start if the user pastes a `dp.*` token. Otherwise ignore.
>
> **Pair with `z-container-kit`** (https://github.com/super-z-kits/z-container-kit) — load both at session start. z-container governs persistence (git IS the disk here — `.env` is committed by design, see z-container law 9); this kit governs the Doppler vault. The two compose: the Doppler PT lives in `/home/user_skills/zk-doppler.env` (NOT the committed `.env`); secrets you fetch FROM Doppler for runtime use are not committed (they live in Doppler, your repo only references them by name).

## Handover (what the user pastes)

```
(Doppler)
- PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- project: example-project
- config: dev | stg | prd
```

**Convention (audit F2):** the `config:` field in the handover MUST be the config that actually contains the seed secrets. If `dev` is empty in your Doppler project but `prd` has the seeds, the handover should say `config: prd` (or you should mirror the seeds into `dev` so the handover works as written). Empty-config false-positives are the most common onboarding failure.

First-time setup (if no Doppler project yet) — create project + 3 configs (dev/stg/prd), write the seeds into the config you'll reference in the handover, return the handover block above for the user to save.

**Config naming (audit F9):** the standard Doppler convention is `dev | stg | prd`. Some kits add `dev_personal` (a per-user sandbox inside a shared project so multiple users can test without clobbering each other's seeds). If your project uses `dev_personal`, document its purpose; otherwise delete it to avoid confusing agents.

## The 5 facts that aren't in training data

1. **Fetch all secrets**: `GET https://api.doppler.com/v3/configs/config/secrets?project=<P>&config=<C>` with `Authorization: Bearer <PT>`. Secret value is at `.secrets.<NAME>.computed` (plain string).

   *(Audit F7: the response also includes `.raw` for each secret, but `.raw` is empty/null on the list endpoint for Doppler's auto-injected `DOPPLER_CONFIG`/`DOPPLER_ENVIRONMENT`/`DOPPLER_PROJECT` metadata keys. Always use `.computed` — never `.raw`.)*

2. **Use the PT directly for the chat session.** It's already in chat (burned). Don't mint a Service Token as a "least-privilege" reflex — the agent sees the PT anyway, so the ST doesn't reduce blast radius. Minting ceremony is theater for the chat session.

3. **Don't put `dp.pt.*` in persistent deployment targets** (GitHub repo secrets, Cloudflare Worker secrets, Vercel/Netlify env vars) — anyone with `repo` scope on GitHub can read a stored secret, and from a `dp.pt.*` they can mint write-capable Service Tokens. Mint a `dp.st.*` Service Token instead (scoped to one project+config, read-only by default, TTL-bounded via `expire_at` Unix ts which auto-revokes + auto-deletes the slug at expiry). See `SKILL-DEPLOY.md` for the deployment flow.

4. **Store the PT in `/home/user_skills/zk-doppler.env` (mode 0600)** — NOT in the project's `.env` and NOT in a shell var. Reasons: (a) `/home/user_skills/` is OUTSIDE the project repo, so git/trufflehog/gitleaks/GitHub push protection can't see it — code reviewers won't flag it and GitHub won't auto-revoke a `ghp_*` if it leaks there; (b) it survives recycles + force-kills (PolarFS) and probably survives into a new chat (per-user namespace, see z-container persistence map); (c) mode 0600 means other processes on the box can't read it; (d) matches the z-container `zk-remote.url` pattern — same place, same posture. Canonical file contents:
   ```
   DOPPLER_PT=dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   DOPPLER_PROJECT=example-project
   DOPPLER_CONFIG=dev
   ```

   **Writing the file (audit F4):** the Super Z `Write` tool can only write under `/home/z/*` — it will refuse `/home/user_skills/...`. Use a bash heredoc instead:
   ```bash
   cat > /home/user_skills/zk-doppler.env <<'EOF'
   DOPPLER_PT=dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   DOPPLER_PROJECT=example-project
   DOPPLER_CONFIG=dev
   EOF
   chmod 0600 /home/user_skills/zk-doppler.env
   ```
   (The bash tool redacts `dp.*` / `ghp_*` / `cfat_*` / `sbp_*` prefixes in display output — fact #5 — so the value won't appear in the transcript.)

   **PT format validation (audit F1):** before the first Doppler API call, sanity-check the PT format:
   ```bash
   printf '%s' "$DOPPLER_PT" | grep -qE '^dp\.(pt|st)\.[A-Za-z0-9]{32,80}$' \
     || { echo "PT looks malformed (length ${#DOPPLER_PT}) — paste artifact?"; exit 1; }
   ```
   A trailing `)` or extra whitespace from chat-client paste is the most common cause. Don't waste an API call on a malformed PT.

   Source at the start of each Bash call that needs it: `set -a; source /home/user_skills/zk-doppler.env; set +a` (each Bash toolcall is a fresh subshell — don't assume vars persist across calls; safest pattern is to source + fetch + use inside a single call, as fact #5 shows). Secrets you FETCH from Doppler for runtime use (GH_PAT, STRIPE_KEY, etc.) go in shell vars only — they're already in Doppler permanently, your repo references them by name, never writes their values to a tracked file. POST `/configs/config/secrets` echoes raw values back at `.secrets.<K>.raw` in the response body — pipe straight to `/dev/null` or capture in a shell var only, don't write the response to a file you'll commit.

5. **Super Z bash tool redacts known token prefixes (`ghp_*`, `dp.*`, `cfat_*`, `sbp_*`) in display output** — the response contains the real value, you just can't SEE it. Don't waste calls chasing "Doppler is restricting the secret." Capture to a shell var and pipe straight to the next call: `GH_PAT=$(curl -s -H "Authorization: Bearer $PT" "https://api.doppler.com/v3/configs/config/secrets?project=$P&config=$C" | jq -r '.secrets.GH_PAT.computed')` then `curl -H "Authorization: Bearer $GH_PAT" ...`. Verify by length: `echo "${#GH_PAT}"`.

🚨 **Don't revoke the user's Personal Token.** It's their master key — they rotate it themselves via the Doppler dashboard after the session. If you accidentally revoke it (e.g. via `DELETE /v3/workplace/personal_tokens/...`), the user loses access to their vault and you've made a mess.

## Smoke test (audit F8)

After writing `/home/user_skills/zk-doppler.env`, run the one-shot smoke test from z-container-kit:

```
bash /home/z/my-project/scripts/zdoppler-smoke
```

It: (a) validates the PT format (audit F1), (b) fetches the secrets, (c) prints `name: computed_len` for each non-DOPPLER_ key, (d) exits 0 if healthy, non-zero if the PT is malformed or the config is empty. Drops the agent's verification step from ~15 lines of boilerplate to a single command. If the kit's `zdoppler-smoke` isn't installed (e.g. older kit copy), the verification boilerplate is:

```bash
set -a; source /home/user_skills/zk-doppler.env; set +a
# Validate format (audit F1)
printf '%s' "$DOPPLER_PT" | grep -qE '^dp\.(pt|st)\.[A-Za-z0-9]{32,80}$' \
  || { echo "PT malformed (len ${#DOPPLER_PT})"; exit 1; }
# Fetch + list (names + lengths only — never print values)
curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
  "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=$DOPPLER_CONFIG" \
  | jq -r '.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not)) | .[] | "\(.key)  (computed len: \(.value.computed | length // 0))"'
```

## Per-provider verification recipes (audit F3)

When you fetch seed credentials from Doppler (e.g. `GH_PAT`, `CF_ACCOUNT_API_KEY`, `SUPABASE_TOKEN`) and want to verify they work, use the RIGHT verification endpoint per provider. The "obvious" /docs endpoint sometimes lies:

**GitHub**: `GET /user` with `Authorization: Bearer <GH_PAT>` → 200 + `{login, ...}` means valid. `GET /repos/<owner>/<repo>` → 200 means accessible; 404 means not visible (token lacks scope OR repo doesn't exist; can't distinguish without trying another repo).

**Cloudflare (audit F3 — the trap)**: `cfat_*`-prefixed API tokens look like they should verify with `POST /user/tokens/verify` — and they DO if the token has User scope. But narrowly-scoped account-only tokens return 401 "Invalid API Token" from `/verify` even though they're perfectly valid. **Use `GET /accounts` instead**: 200 + non-empty `result` = token valid. `/user/tokens/verify` is only authoritative for tokens that carry the "User API Tokens: Read" scope.

**Supabase**: `GET /v1/projects` → 200 + array (possibly empty) = token valid. `GET /v1/organizations` → 200 + array of orgs the token can see.

## Situational specifics (rare ops)

**Reading one secret by name** (instead of fetching all): `GET /v3/configs/config/secret?name=X` (singular + query param). NOT `/secrets/{NAME}` — that 404s. Value at `.value.computed`.

**Writing a secret** (only with the user's explicit ask): `POST /v3/configs/config/secrets` with body `{"project":"X","config":"Y","secrets":{"KEY":"value"}}` (flat string values, not nested — nested returns 400 "must be a string"). POST echoes raw values back at `.secrets.<K>.raw` — pipe to `/dev/null`, don't capture to a committable file.

**Mirroring seeds across configs (audit F2 fix)**: if the user's handover says `config: dev` but the seeds are in `prd`, the cleanest fix is to mirror them. Fetch from `prd`, POST to `dev`:
```bash
set -a; source /home/user_skills/zk-doppler.env; set +a
SEEDS=$(curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
  "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=prd")
# Build a flat-string body from prd's non-DOPPLER_ secrets
BODY=$(printf '%s' "$SEEDS" | jq -c --arg p "$DOPPLER_PROJECT" '{project:$p, config:"dev",
  secrets: (.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not))
            | map({(.key): .value.computed}) | add)}')
# POST to dev, discard the response (it echoes raw values)
curl -sS -o /dev/null -X POST -H "Authorization: Bearer $DOPPLER_PT" \
  -H "Content-Type: application/json" -d "$BODY" \
  "https://api.doppler.com/v3/configs/config/secrets"
```

**Deleting a secret**: `DELETE /v3/configs/config/secret?project=X&config=Y&name=K` (no body, HTTP 204). Reading a deleted secret returns 200 + null `value` fields, not 404.

## When you need to read SKILL-DEPLOY.md

If the task involves deploying to a persistent target (GitHub Actions workflow, Cloudflare Worker, Supabase Edge Function, Vercel/Netlify) and wiring secrets into it — read [`SKILL-DEPLOY.md`](./SKILL-DEPLOY.md) in this repo. It covers:
- Minting a `dp.st.*` Service Token for the deployment target (the one legitimate ST use case)
- GH Actions pull-at-runtime pattern (`doppler run` in the workflow)
- Cloudflare Workers DIY pull (Doppler has no auto-sync to Workers — only Pages)
- Supabase one-click auto-sync OR manual Management API
- Rotation mechanics per platform (chain-reaction resolved via seed/worker split)

Don't read SKILL-DEPLOY.md for: reading secrets to use in chat, fetching a token to call an API directly, one-shot tasks, or local dev work.

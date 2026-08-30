# secrets-vault-skill

> Read this at session start if the user pastes a `dp.*` token. Otherwise ignore.
>
> **Pair with `z-container-kit`** — load both at session start, reading the
> LOCAL copies (web fallback only if a local copy is absent:
> `https://raw.githubusercontent.com/super-z-kits/<repo>/main/<file>`).
> z-container governs persistence and git (`.env` is committed by design —
> its law 9); this kit governs the Doppler vault: the PT lives in
> `/home/user_skills/${ZK_PREFIX}-doppler.env` (a sanctioned user_skills
> write — fact #4), never in the committed `.env`. ZK_PREFIX resolves like
> z-container's helpers (`ZK_PREFIX` env > `$PROJ/.agents/config`; each
> bash toolcall is a fresh subshell). This kit's tool: `bash
> /home/user_skills/secrets-vault-kit/scripts/zdoppler-smoke`; update its
> installed copy via the README copy-then-swap (never rm -rf a live copy —
> a parallel session may be reading it).

## Handover (what the user pastes)

```
(Doppler)
- PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- project: example-project
- config: dev | stg | prd
```

**The `config:` field MUST name the config that actually contains the seed
secrets.** If `dev` is empty but `prd` has the seeds, the handover should
say `config: prd` (or mirror the seeds into `dev`). Empty-config
false-positives are the most common onboarding failure.

First-time setup (if no Doppler project yet) — create project + 3 configs
(dev/stg/prd), write the seeds into the config you'll reference in the
handover, return the handover block above for the user to save.

**`dev_personal` configs:** the standard convention is `dev | stg | prd`;
`dev_personal` is a per-user sandbox inside a shared project. If you
encounter one that isn't documented as intentional, flag it to the user —
don't assume it's safe to use as your config. Intentional ones are
documented in the project description; an ownerless leftover → recommend
deleting it (Doppler dashboard).

## The 5 facts that aren't in training data

1. **Fetch all secrets**: `GET https://api.doppler.com/v3/configs/config/secrets?project=<P>&config=<C>` with `Authorization: Bearer <PT>`. Secret value is at `.secrets.<NAME>.computed` (plain string). Always use `.computed` — `.raw` is empty/null on the list endpoint for Doppler's auto-injected `DOPPLER_*` metadata keys.

2. **Use the PT directly for the chat session.** It's already in chat (burned). Don't mint a Service Token as a "least-privilege" reflex — the agent sees the PT anyway, so the ST doesn't reduce blast radius. Minting ceremony is theater for the chat session.

3. **Don't put `dp.pt.*` in persistent deployment targets** (GitHub repo secrets, Cloudflare Worker secrets, Vercel/Netlify env vars) — anyone with `repo` scope on GitHub can read a stored secret, and from a `dp.pt.*` they can mint write-capable Service Tokens. Mint a `dp.st.*` Service Token instead (scoped to one project+config, read-only by default, TTL-bounded via `expire_at` Unix ts which auto-revokes + auto-deletes the slug at expiry). See `SKILL-DEPLOY.md` for the deployment flow.

4. **Store the PT in `/home/user_skills/${ZK_PREFIX}-doppler.env` (mode 0600)** — NOT in the project's `.env` and NOT in a shell var. It sits outside the repo (invisible to git/trufflehog/gitleaks and GitHub push protection — no reviewer flags, no auto-revoke), on PolarFS (survives recycles + force-kills, probably new chats), at 0600. Canonical contents:
   ```
   DOPPLER_PT=dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   DOPPLER_PROJECT=example-project
   DOPPLER_CONFIG=dev
   DOPPLER_PT_STORED_AT=2026-08-30T00:00:00Z
   ```

   **Writing the file:** the Super Z `Write` tool refuses `/home/user_skills/...` — use bash. `/home/user_skills` is shared across concurrent chats with no git (z-container's static rule), so the write must be atomic (same-dir tmp + `mv`) and fresher-wins (an older paste never clobbers a newer stored rotation from a parallel session). With the handover values in shell vars:
   ```bash
   F=/home/user_skills/${ZK_PREFIX}-doppler.env
   : "${DOPPLER_PT:?set DOPPLER_PT from the handover paste}" \
     "${DOPPLER_PROJECT:?}" "${DOPPLER_CONFIG:?}"   # fail loudly, never write empty values
   NEW_TS=$(date -u +%Y-%m-%dT%H:%M:%SZ)
   OLD_TS=$(sed -n 's/^DOPPLER_PT_STORED_AT=//p' "$F" 2>/dev/null)
   if [ -n "$OLD_TS" ] && [ "$OLD_TS" \> "$NEW_TS" ]; then
     echo "keeping existing file (stored $OLD_TS is NEWER — a parallel session already rotated)"
   else
     T="$F.tmp.$$"
     printf 'DOPPLER_PT=%s\nDOPPLER_PROJECT=%s\nDOPPLER_CONFIG=%s\nDOPPLER_PT_STORED_AT=%s\n' \
       "$DOPPLER_PT" "$DOPPLER_PROJECT" "$DOPPLER_CONFIG" "$NEW_TS" > "$T"
     chmod 0600 "$T" && mv -f "$T" "$F"   # atomic swap: readers see old-or-new, never partial
   fi
   ```
   (No heredoc on purpose — an indented `EOF` terminator breaks a verbatim
   copy-paste. printf is indent-immune. Never echo real token values.)

   The stored PT is stale-by-policy (the user rotates after every chat);
   `DOPPLER_PT_STORED_AT` is how you know how stale, and `zdoppler-smoke`
   warns if it's older than 7 days.

   **PT format validation** — before the first Doppler API call (a trailing
   `)` or extra whitespace from chat-client paste is the most common cause):
   ```bash
   printf '%s' "$DOPPLER_PT" | grep -qE '^dp\.(pt|st)\.[A-Za-z0-9]{32,80}$' \
     || { echo "PT looks malformed (length ${#DOPPLER_PT}) — paste artifact?"; exit 1; }
   ```

   Source the file at the start of each bash call that needs it: `set -a;
   source /home/user_skills/${ZK_PREFIX}-doppler.env; set +a`. Secrets you
   FETCH from Doppler for runtime use (GH_PAT, STRIPE_KEY, …) go in shell
   vars only — they're already in Doppler permanently; your repo references
   them by name, never writes their values to a tracked file. POST
   `/configs/config/secrets` echoes raw values back at `.secrets.<K>.raw` —
   pipe straight to `/dev/null` or capture in a shell var only.

   **Staging pattern for multi-call flows** (verification flows that span
   multiple bash calls): stage the fetched secrets as a 0600 JSON file under
   `/tmp/my-project/` (per-chat, untracked, outside the git tree):
   ```bash
   set -a; source /home/user_skills/${ZK_PREFIX}-doppler.env; set +a
   curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
     "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=$DOPPLER_CONFIG" \
     | jq '.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not)) | map({(.key): .value.computed}) | add' \
     > /tmp/my-project/doppler-secrets.json
   chmod 0600 /tmp/my-project/doppler-secrets.json
   GH_PAT=$(jq -r .GH_PAT /tmp/my-project/doppler-secrets.json)
   # ... use $GH_PAT ... ; at session end: rm -f /tmp/my-project/doppler-secrets.json
   ```
   Single-secret one-shots still use the pipe-through pattern (fact #5).

5. **The bash tool redactor is PREFIX-SELECTIVE, not comprehensive.** Observed 2026-08-28: only `ghp_*` is masked (`[REDACTED:github_token]`); `dp.pt.*`, `dp.st.*`, `cfat_*`, and `sbp_*` print in CLEARTEXT in all output. This may change with platform builds — verify at session start with realistic-length fakes (too-short fakes like `ghp_aaaa` don't trigger the redactor):
   ```bash
   for t in "ghp_$(printf 'a%.0s' {1..36})" "dp.pt.$(printf 'a%.0s' {1..43})" \
            "dp.st.$(printf 'a%.0s' {1..43})" "cfat_$(printf 'a%.0s' {1..48})" \
            "sbp_$(printf 'a%.0s' {1..40})"; do
     printf '%s\n' "$t"; done   # any line printing cleartext = that prefix is NOT redacted
   ```
   Given the redactor is unreliable: **never `cat`/`head`/`echo` a real secret value**. Capture to a shell var and pipe straight to the next call: `GH_PAT=$(curl ... | jq -r '.secrets.GH_PAT.computed')` then `curl -H "Authorization: Bearer $GH_PAT" ...`. Verify by length only: `echo "${#GH_PAT}"`.

🚨 **Don't revoke the user's Personal Token.** It's their master key — they rotate it themselves via the Doppler dashboard after the session. If you accidentally revoke it (e.g. via `DELETE /v3/workplace/personal_tokens/...`), the user loses access to their vault and you've made a mess.

## Smoke test

After writing `/home/user_skills/${ZK_PREFIX}-doppler.env`:

```
bash /home/user_skills/secrets-vault-kit/scripts/zdoppler-smoke
```

It validates the PT format, fetches the secrets, prints `name: computed_len`
for each non-DOPPLER_ key, and exits 0 if healthy (non-zero if the PT is
malformed or the fetch fails; an empty config exits 0 with a ⚠️ warning) —
drops ~15 lines of verification
boilerplate to one command. It resolves the prefix like z-container's
helpers and accepts `--env-file` / `--project` / `--config` overrides. If
the script is unavailable, the verification boilerplate:

```bash
set -a; source /home/user_skills/${ZK_PREFIX}-doppler.env; set +a
printf '%s' "$DOPPLER_PT" | grep -qE '^dp\.(pt|st)\.[A-Za-z0-9]{32,80}$' \
  || { echo "PT malformed (len ${#DOPPLER_PT})"; exit 1; }
curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
  "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=$DOPPLER_CONFIG" \
  | jq -r '.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not)) | .[] | "\(.key)  (computed len: \(.value.computed | length // 0))"'
```

## Vault-sourced GitHub bootstrap (Path C)

If the user does NOT paste a GitHub PAT — the GH_PAT lives inside the
Doppler vault (e.g. `agent-bootstrap` project; that name is the hint this
is the intended flow) — on a true cold start (no origin wired, no account
default, no PAT pasted):

```bash
# 1. Write the Doppler env file (per fact #4)
# 2. Confirm the vault is reachable + GH_PAT is present
bash /home/user_skills/secrets-vault-kit/scripts/zdoppler-smoke
# 3. Stage the vault secrets (staging pattern above)
set -a; source /home/user_skills/${ZK_PREFIX}-doppler.env; set +a
curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
  "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=$DOPPLER_CONFIG" \
  | jq '.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not)) | map({(.key): .value.computed}) | add' \
  > /tmp/my-project/doppler-secrets.json
chmod 0600 /tmp/my-project/doppler-secrets.json
# 4. Extract GH_PAT and wire the remote (PAT never echoed — fact #5)
GH_PAT=$(jq -r .GH_PAT /tmp/my-project/doppler-secrets.json)
git -C /home/z/my-project remote add origin "https://${GH_PAT}@github.com/<user>/<repo>.git"
git -C /home/z/my-project fetch origin
git -C /home/z/my-project log origin/main --oneline -5   # SANITY CHECK before reset
git -C /home/z/my-project reset --hard origin/main
bash /home/user_skills/z-container-kit/scripts/zk-init <name>
bash /home/user_skills/z-container-kit/scripts/zsave "fresh-chat vault-sourced bootstrap checkpoint"
rm -f /tmp/my-project/doppler-secrets.json   # cleanup staged secrets
```

## Per-provider verification recipes

When you fetch seed credentials from Doppler and want to verify they work,
use the RIGHT endpoint per provider — the "obvious" /docs endpoint
sometimes lies:

**GitHub**: `GET /user` with `Authorization: Bearer <GH_PAT>` → 200 + `{login, ...}` means valid. `GET /repos/<owner>/<repo>` → 200 means accessible; 404 means not visible (token lacks scope OR repo doesn't exist; can't distinguish without trying another repo).

**Cloudflare**: `cfat_*`-prefixed API tokens look like they should verify with `POST /user/tokens/verify` — and they DO if the token has User scope. But narrowly-scoped account-only tokens return 401 "Invalid API Token" from `/verify` even though they're perfectly valid. **Use `GET /accounts` instead**: 200 + non-empty `result` = token valid. `/user/tokens/verify` is only authoritative for tokens that carry the "User API Tokens: Read" scope.

**Supabase**: `api.supabase.com` is fronted by a Cloudflare WAF. `python-urllib`'s default UA (`Python-urllib/3.x`) → HTTP 403 with EMPTY body (or `error code: 1010`) — reads exactly like an auth failure. **Always send a custom `User-Agent`** (e.g. `User-Agent: ${ZK_PREFIX}-verify`). With curl this is automatic; with urllib/requests/axios, set it explicitly. Once you have 200: `GET /v1/projects` → 200 + array (possibly empty `[]` = token valid, account has no projects — NOT an error). `GET /v1/organizations` → 200 + array of orgs the token can see.

## Situational specifics (special-case operations)

**Reading one secret by name** (instead of fetching all): `GET /v3/configs/config/secret?name=X` (singular + query param). NOT `/secrets/{NAME}` — that 404s. Value at `.value.computed`.

**Writing a secret** (only with the user's explicit ask): `POST /v3/configs/config/secrets` with body `{"project":"X","config":"Y","secrets":{"KEY":"value"}}` (flat string values, not nested — nested returns 400 "must be a string"). POST echoes raw values back at `.secrets.<K>.raw` — pipe to `/dev/null`.

**Mirroring seeds across configs**: if the handover says `config: dev` but the seeds are in `prd`, fetch from `prd`, POST to `dev`:
```bash
set -a; source /home/user_skills/${ZK_PREFIX}-doppler.env; set +a
SEEDS=$(curl -sS -H "Authorization: Bearer $DOPPLER_PT" \
  "https://api.doppler.com/v3/configs/config/secrets?project=$DOPPLER_PROJECT&config=prd")
BODY=$(printf '%s' "$SEEDS" | jq -c --arg p "$DOPPLER_PROJECT" '{project:$p, config:"dev",
  secrets: (.secrets | to_entries | map(select(.key | startswith("DOPPLER_") | not))
            | map({(.key): .value.computed}) | add)}')
curl -sS -o /dev/null -X POST -H "Authorization: Bearer $DOPPLER_PT" \
  -H "Content-Type: application/json" -d "$BODY" \
  "https://api.doppler.com/v3/configs/config/secrets"
```

**Listing configs in a project**: use `GET /v3/configs?project=<P>` (slug-as-query-param). Do NOT use `GET /v3/projects/project/<P>` (slug-in-path) — that returns 400 with `{"messages":["You must specify a project"],"success":false}` even though the project IS specified. The configs response keys on `name`/`slug`(UUID)/`root`/`locked`/`environment` — NOT `id`.

**Workplace / PT identity**: `GET /v3/workplace` with the PT Bearer returns the workplace name/slug/id — a PT-validity check that doesn't require knowing the project name.

**Deleting a secret**: `DELETE /v3/configs/config/secret?project=X&config=Y&name=K` (no body, HTTP 204). Reading a deleted secret returns 200 + null `value` fields, not 404.

## When you need to read SKILL-DEPLOY.md

If the task involves deploying to a persistent target (GitHub Actions
workflow, Cloudflare Worker, Supabase Edge Function, Vercel/Netlify) and
wiring secrets into it — read `SKILL-DEPLOY.md` (minting the deployment
target's `dp.st.*`, per-platform pull patterns, rotation mechanics). Don't
read it for: reading secrets to use in chat, one-shot API calls, or local
dev work.

## Token fingerprint table

When you encounter an unknown token, identify it by prefix + length:

| Prefix | Provider | Length | Auth scheme | Notes |
|---|---|---|---|---|
| `ghp_` | GitHub PAT (classic) | 40 | `Authorization: Bearer <token>` | deprecated 2024-ish but still common; rotate to `github_pat_` |
| `github_pat_` | GitHub PAT (fine-grained) | 50-90 | `Authorization: Bearer <token>` | replaces `ghp_`; per-repo scopeable |
| `dp.pt.` | Doppler Personal Token | ~49 | `Authorization: Bearer <token>` | user master key; mint STs from this |
| `dp.st.` | Doppler Service Token | ~49 | `Authorization: Bearer <token>` | scoped to one project+config; TTL-bounded via `expire_at` |
| `cfat_` | Cloudflare API Token | 40-100 | `Authorization: Bearer <token>` | per-token scope; verify via `GET /accounts` (not `/user/tokens/verify`) |
| `sbp_` | Supabase Access Token | ~44 | `Authorization: Bearer <token>` | from supabase.com/dashboard/account/tokens; send custom UA |

All lengths observed 2026-08; individual tokens may vary by a few chars. The bash display redactor is PREFIX-SELECTIVE (see fact #5) — never pipe a real token through echo/cat/printf; verify by length, not by sight.

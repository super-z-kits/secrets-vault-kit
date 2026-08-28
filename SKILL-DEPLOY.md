# SKILL-DEPLOY.md — deployment-specific practices

> Load this file ONLY when the task involves deploying to a persistent target (GitHub Actions, Cloudflare Workers, Supabase, Vercel/Netlify) and wiring secrets into it. For everything else, `SKILL.md` is enough.

## The one rule that's different in deployment

When you put a Doppler credential into a deployment target (GitHub repo secret, Cloudflare Worker secret, Vercel/Netlify env var), it must be a `dp.st.*` Service Token — **never** a `dp.pt.*` Personal Token.

Reason: anyone with `repo` scope on GitHub can read a stored secret. If it's a `dp.pt.*`, they can mint write-capable Service Tokens from it → full vault compromise. A `dp.st.*` is scoped to one (project, config), read-only by default, and TTL-bounded via `expire_at` — so even if it leaks, the blast radius is one config and the TTL.

🚨 The user's `dp.pt.*` Personal Token is their master key. Don't put it in any deployment target. Don't revoke it either — they rotate it themselves via the Doppler dashboard after each session.

## Mint a Service Token for a deployment target

```bash
PT=dp.pt.xxxxx; PROJ=my-app; CFG=prd
EXPIRE_AT=$(($(date +%s) + 604800))   # 7d; use a long TTL for deployment tokens (the user will rotate by re-minting)

curl -fsS -X POST -H "Authorization: Bearer $PT" -H "Content-Type: application/json" \
  -d "{\"project\":\"$PROJ\",\"config\":\"$CFG\",\"name\":\"deploy-$(date +%Y%m%d)\",\"access\":\"read\",\"expire_at\":$EXPIRE_AT}" \
  https://api.doppler.com/v3/configs/config/tokens
# → {"token":{"key":"dp.st.prd.xxxx...","slug":"<UUID>","expires_at":"<ISO>"}}
```

Capture `token.key` — that's what you put in the deployment target's secret store. Store it in `/home/user_skills/zk-deploy-tokens.env` (mode 0600) — outside the project repo, so git/scanners/reviewers can't see it (see SKILL.md fact #4 for the full rationale). In a normal server environment, use a `chmod 600` file or secret manager (env vars leak to `/proc/<pid>/environ`).

To revoke later (e.g. user asks to rotate the deploy token):

```bash
curl -fsS -X DELETE -H "Authorization: Bearer $PT" -H "Content-Type: application/json" \
  -d "{\"project\":\"$PROJ\",\"config\":\"$CFG\",\"slug\":\"$SLUG\"}" \
  https://api.doppler.com/v3/configs/config/tokens/token
# → HTTP 200 + {"success":true}
```

(Re-mint a new one first, update the deployment target, verify it works, THEN revoke the old one — zero downtime.)

## GitHub Actions — pull at runtime (recommended)

This pattern uses Doppler as the source of truth; no secrets are duplicated in GitHub. The repo holds ONE secret — a `dp.st.*` Service Token (long TTL or rotated quarterly).

```yaml
# .github/workflows/deploy.yml
name: Deploy
on: { push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dopplerhq/cli-action@v4
      - name: Run with secrets injected as env vars
        env:
          DOPPLER_TOKEN: ${{ secrets.DOPPLER_TOKEN }}     # dp.st.* — never dp.pt.*
          DOPPLER_PROJECT: my-app
          DOPPLER_CONFIG: prd
        run: doppler run --command="npm run deploy"
```

Store `DOPPLER_TOKEN` once via `gh secret set DOPPLER_TOKEN` (if you have the `gh` CLI) OR via the GitHub REST API (libsodium sealed-box encryption):

```bash
# 1. Get the repo's public key (one per repo)
curl -fsS -H "Authorization: Bearer $GH_PAT" \
  https://api.github.com/repos/$OWNER/$REPO/actions/secrets/public-key \
  | jq -r '"\(.key_id) \(.key)"' | read KEY_ID KEY_B64

# 2. Encrypt the DOPPLER_TOKEN (dp.st.*) with libsodium sealed-box
#    PyNaCl gotcha: use RawEncoder for the public key (the b64-decoded bytes), NOT Base64Encoder
python3 -m pip install pynacl >/dev/null 2>&1
ENCRYPTED=$(python3 -c "
import base64
from nacl import public, encoding
pubkey_bytes = base64.b64decode('$KEY_B64')
pub = public.PublicKey(pubkey_bytes, encoder=encoding.RawEncoder)   # NOT Base64Encoder
sealed = public.SealedBox(pub).encrypt(b'$DOPPLER_TOKEN_ST')
print(base64.b64encode(sealed).decode())
")

# 3. PUT the encrypted secret
curl -fsS -X PUT -H "Authorization: Bearer $GH_PAT" -H "Content-Type: application/json" \
  -d "{\"encrypted_value\":\"$ENCRYPTED\",\"key_id\":\"$KEY_ID\"}" \
  https://api.github.com/repos/$OWNER/$REPO/actions/secrets/DOPPLER_TOKEN
```

Note: the workflow above assumes the repo has a `package.json` with a `deploy` script. If it doesn't, the `npm run deploy` step will fail with "missing script: deploy" — add a minimal `package.json` first.

**Alternative** — Doppler auto-sync: dashboard → Integrations → GitHub → connect. Doppler pushes all secrets to GH repo/org secrets automatically. Use this if you want them visible in the GH UI; otherwise pull-at-runtime is the recommended default (single source of truth = Doppler).

## Cloudflare Workers — DIY pull

Doppler has NO auto-sync to Workers (only to Cloudflare Pages). You must pull secrets in your CI and push them to the Worker:

```yaml
- uses: dopplerhq/cli-action@v4
- name: Push secrets to Worker
  env:
    DOPPLER_TOKEN: ${{ secrets.DOPPLER_TOKEN }}     # dp.st.*
    DOPPLER_PROJECT: my-app
    DOPPLER_CONFIG: prd
  run: |
    doppler secrets --json \
      | jq -c 'with_entries(.value = .value.computed)' \
      | wrangler secret bulk
- uses: cloudflare/wrangler-action@v4
  with:
    apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}    # cfat_* — the one unavoidable long-lived cloud credential
    accountId: ${{ secrets.CF_ACCOUNT_ID }}
    command: deploy
```

`wrangler secret bulk` reads a flat JSON `{KEY:VALUE}` from stdin. `cloudflare/wrangler-action` does NOT support GitHub OIDC — you must pass `apiToken`. This is the one unavoidable long-lived cloud credential in the stack.

## Supabase Edge Functions — one-click auto-sync (recommended) OR manual

**Auto-sync** (recommended): Doppler dashboard → Integrations → Supabase → connect with a Supabase access token. Doppler pushes to the project's Edge Function secrets automatically. Updates are live immediately (no redeploy needed).

**Manual API** (when you have the `sbp_*` Supabase PAT):

```bash
curl -fsS -X POST -H "Authorization: Bearer $SUPABASE_PAT" -H "Content-Type: application/json" \
  -d '{"secrets":[{"name":"STRIPE_KEY","value":"sk_live_xxx"}]}' \
  https://api.supabase.com/v1/projects/$REF/secrets
```

⚠️ Supabase secret names must NOT start with `SUPABASE_` (reserved). Max 256 chars name, 24576 value. Live immediately, no redeploy.

## Rotation playbook

The user's original concern: "I paste a key in chat → session ends → key is burned → but the deployed service still uses the same key → rotation is a chain reaction." Here's the resolution:

| What | When to rotate | How + impact on deployed services |
|---|---|---|
| Doppler Personal Token (`dp.pt.*`) | After every chat session where it was pasted in plaintext. | User does it via Doppler dashboard (not the agent — don't revoke the master key). Zero impact on deployed services — they use Service Tokens, not the Personal Token. |
| Doppler Service Token (`dp.st.*`) used by deployments | Quarterly, on suspicion, or on team turnover. | Agent mints new via API (new `expire_at`), updates the deployment target's secret with the new value, verifies the deployment still works, then revokes old. ~30 sec, zero downtime. |
| Cloud provider tokens (`cfat_*`, `ghp_*`, `sbp_*`) | Quarterly, or eliminate via OIDC where possible. | Mint new, store the new value in Doppler under `CF_API_TOKEN` / `GH_PAT` / etc. Doppler auto-sync propagates to all consumers; redeploy to pick up. Delete old. |
| App secrets (Stripe key, DB URL, etc.) | Per the secret's own policy. | Update once in Doppler. Auto-sync targets get the new value immediately; running services pick it up on next fetch (CF Workers Secrets Store / Supabase Edge Functions = live; GH Actions = next run reads fresh). |

**The chain-reaction doesn't happen** because of the seed/worker split: deployed services use Service Tokens (which have their own lifecycle, independent of the Personal Token). Rotating the chat-shared Personal Token after a session is a one-click user action with zero production impact.

**Use `expire_at` for the deployment Service Token too** — even if it's far in the future (90 days), it forces a re-mint cycle, which prevents the credential from going stale. If the agent forgets to rotate, the TTL catches it.

## When NOT to load this file

- Reading secrets to use in chat (covered by SKILL.md).
- Fetching a token to call an API directly (covered by SKILL.md).
- One-shot tasks, local dev work (covered by SKILL.md).
- The user explicitly says "just fetch X from Doppler and use it" — no deployment target involved.

If in doubt: SKILL.md covers the daily workflow. Load this file only when you're about to put a credential in a persistent target OR the user explicitly asks about deployment patterns.

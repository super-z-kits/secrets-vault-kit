# secrets-vault-kit

> Operational reflex for an AI agent (Claude / Cursor / Codex / GLM / Super Z / etc.) working with a user's Doppler-based secrets on a Next.js / Cloudflare / Supabase / GitHub stack. Pair with [`z-container-kit`](https://github.com/super-z-kits/z-container-kit).

## Two files, lazy-loaded

- **`SKILL.md`** (45 lines) — always loaded when the user pastes a `dp.*` token. 5 facts + situational specifics + "When you need DEPLOY.md" gating.
- **`DEPLOY.md`** (151 lines) — lazy-read this only when the task involves persistent deployment targets (GH Actions, CF Workers, Supabase, Vercel/Netlify). Mint a `dp.st.*` Service Token for the deployment target + per-platform deploy patterns + rotation playbook.

## Seed credential (example of what the user pastes)

```
Doppler PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
project: example-project
config: dev
<task description>
```

The PT is the seed; everything else is fetched from Doppler at runtime.

## The 5 facts (the load-bearing value-add)

1. Fetch endpoint + `.computed` path + DOPPLER_* auto-inject gotcha
2. **Use the PT directly for the chat session** — don't mint a Service Token as a "least-privilege" reflex (theater; the PT is already in chat, so the ST doesn't reduce blast radius)
3. Only mint an ST for persistent deployment targets (privilege-escalation protection) + `expire_at` mechanism
4. **Store the PT in `/home/user_skills/zk-doppler.env` (mode 0600)** — outside the project repo, so git/trufflehog/gitleaks/GitHub push protection can't see it. Code reviewers won't flag it, GitHub won't auto-revoke. Matches the z-container `zk-remote.url` pattern. Secrets you FETCH from Doppler for runtime use go in shell vars only.
5. Super Z bash tool redacts known token prefixes in display — capture to shell var, verify by length, pipe to next call (don't chase "Doppler is restricting the secret")

# secrets-vault-kit

> Operational reflex for an AI agent (Claude / Cursor / Codex / GLM / Super Z / etc.) working with a user's Doppler-based secrets on a Next.js / Cloudflare / Supabase / GitHub stack. Pair with [`z-container-kit`](https://github.com/super-z-kits/z-container-kit).

## Two files, lazy-loaded

- **`SKILL.md`** (45 lines) — always loaded when the user pastes a `dp.*` token. 5 facts + situational specifics + "When you need DEPLOY.md" gating.
- **`DEPLOY.md`** (151 lines) — lazy-loaded only when the task involves persistent deployment targets (GH Actions, CF Workers, Supabase, Vercel/Netlify). Mint a `dp.st.*` Service Token for the deployment target + per-platform deploy patterns + rotation playbook.

## The handover (what the user pastes)

```
Doppler PT: dp.pt.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
project: example-project
config: dev
<task description>
```

No magic word — the `dp.*` token in the user's paste triggers the skill. The PT is the seed; everything else is fetched from Doppler at runtime.

## The 5 facts (the load-bearing value-add)

1. Fetch endpoint + `.computed` path + DOPPLER_* auto-inject gotcha
2. **Use the PT directly for the chat session** — don't mint a Service Token as a "least-privilege" reflex (theater; the PT is already in chat, so the ST doesn't reduce blast radius)
3. Only mint an ST for persistent deployment targets (privilege-escalation protection) + `expire_at` mechanism
4. **Store the PT in `/home/user_skills/zk-doppler.env` (mode 0600)** — outside the project repo, so git/trufflehog/gitleaks/GitHub push protection can't see it. Code reviewers won't flag it, GitHub won't auto-revoke. Matches the z-container `zk-remote.url` pattern. Secrets you FETCH from Doppler for runtime use go in shell vars only.
5. Super Z bash tool redacts known token prefixes in display — capture to shell var, verify by length, pipe to next call (don't chase "Doppler is restricting the secret")

## Pair with z-container-kit

[`z-container-kit`](https://github.com/super-z-kits/z-container-kit) is the survival guide for the Super Z sandbox — persistence model, git HEAD watchdog, recycle-safe worktrees, `.env` practice. **Load both at session start.**

The two kits compose:
- **z-container-kit** governs persistence: git IS the disk here, the watchdog resets to main before every toolcall, worktrees live on PolarFS, credential files live in `/home/user_skills/` (mode 0600) — outside the project repo, so scanners/reviewers/GitHub push protection can't see them.
- **secrets-vault-kit** (this) governs the Doppler vault: how to fetch secrets, when to mint an ST, deployment patterns. The Doppler PT follows the same persistence posture as z-container's `zk-remote.url` — a 0600 file in `/home/user_skills/zk-doppler.env`, NOT in the project's `.env`.

The tension we resolved: an earlier version of this kit followed z-container's "git IS the disk" advice and put the Doppler PT in the committed `.env`. That conflicted with code review (reviewers flag any committed secret as P0) and with GitHub's push protection (which can auto-revoke a `ghp_*` if it's committed). The fix: put the PT in `/home/user_skills/` (same place z-container puts `zk-remote.url`) — outside the project repo, git-invisible by construction, no `.gitignore` dance needed.

## Validation history (8 rounds)

- R1: cold-test + audit (17 issues) + e2e live GH Actions deploy (✅ first try, 15s, all 8 secrets pulled from Doppler).
- R2: verified audit's 2 critical API claims; iterated SKILL.md applying all 17 fixes.
- R3: fresh peer-review subagent — 17/17 fixes verified FIXED. Applied 3 polish items.
- R4: examined a parallel workstream's repo; integrated clearly-better findings (`expire_at`, POST echo warning, two-tier model).
- R5: ruthless line-by-line audit with "response is in LLM context anyway" lens (339 → 240 lines). Dropped security theater (length-only verify, mktemp/chmod 600/shred for transient outputs).
- R5+: tested "do we even need SKILL.md?" with fresh sub-agent. Without SKILL.md, friction 4/5 (3 of those points were an environment-specific bash tool redaction quirk). With minimal SKILL.md, friction 1/5. Cut SKILL.md 240 → 74 lines.
- R5++: polished — dropped magic word, simplified handover labels, added bash-redaction as fact #5.
- R5+++: dropped the mint-default reflex (theater for chat session — PT is already burned, ST doesn't reduce blast radius); reclassified "Beyond the basics" into situational specifics.
- **R5++++ (this round)**: split into SKILL.md (always loaded) + DEPLOY.md (lazy-loaded for deployment tasks). 3-wave usability test (simple vault → dev scenario → deployment) — friction 1/5, 2/5, 2/5. PT verified intact at the end of each wave (the 🚨 "don't revoke the master PT" rule held). Added z-container-kit pairing note + fixed the .env conflict (fact #4 now says the PT goes in the committed `.env` per z-container law 9; secrets fetched from Doppler for runtime use are not committed).

## How this differs from `zikomolapoutl/secrets-agent-kit`

The parallel workstream (slightly newer model) ships 6 files + 3 helper scripts. This kit is 2 files (SKILL.md + DEPLOY.md), no scripts. Same conclusion by independent paths.

Where the two kits contradict on technical findings, this kit kept its own verified findings (per user instruction): the per-secret `/secret?name=X` query endpoint (parallel kit tested only the path-style `/secrets/{NAME}` that 404s); the simpler `access:"read"` mint body field; the fact #2 "don't mint reflexively" rule.

## License

MIT. Use freely.

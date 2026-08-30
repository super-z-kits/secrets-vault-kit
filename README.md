# secrets-vault-kit

> Operational reflex for an AI agent (Claude / Cursor / Codex / GLM / Super Z / etc.) working with a user's Doppler-based secrets on a Next.js / Cloudflare / Supabase / GitHub stack. Pair with [`z-container-kit`](https://github.com/super-z-kits/z-container-kit).

## Two files, lazy-loaded

- **`SKILL.md`** — always loaded when the user pastes a `dp.*` token. 5 facts + situational specifics + "When you need DEPLOY.md" gating.
- **`SKILL-DEPLOY.md`** — lazy-loaded only when the task involves persistent deployment targets (GH Actions, CF Workers, Supabase, Vercel/Netlify). Mint a `dp.st.*` Service Token for the deployment target + per-platform deploy patterns + rotation playbook.

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
4. **Store the PT in `/home/user_skills/${ZK_PREFIX}-doppler.env` (mode 0600)** — outside the project repo, so git/trufflehog/gitleaks/GitHub push protection can't see it. Code reviewers won't flag it, GitHub won't auto-revoke. Same posture as z-container's account default (`zk-default.env` — v5.1 replaced the old `${ZK_PREFIX}-remote.url` files). Secrets you FETCH from Doppler for runtime use go in shell vars only.
5. Super Z bash tool redacts known token prefixes in display — capture to shell var, verify by length, pipe to next call (don't chase "Doppler is restricting the secret")

## Pair with z-container-kit

[`z-container-kit`](https://github.com/super-z-kits/z-container-kit) is the survival guide for the Super Z sandbox — persistence model, git HEAD watchdog, recycle-safe worktrees, `.env` practice. **Load both at session start.**

The two kits compose:
- **z-container-kit** governs persistence: git IS the disk here, the watchdog resets to main before every toolcall, worktrees live on PolarFS, credential files live in `/home/user_skills/` (mode 0600) — outside the project repo, so scanners/reviewers/GitHub push protection can't see them.
- **secrets-vault-kit** (this) governs the Doppler vault: how to fetch secrets, when to mint an ST, deployment patterns. The Doppler PT follows the same persistence posture as z-container's account default — a 0600 file in `/home/user_skills/${ZK_PREFIX}-doppler.env`, NOT in the project's `.env`. Since v2.10 this kit also ships the Doppler smoke tool (`scripts/zdoppler-smoke`, moved from z-container-kit).

The tension we resolved: an earlier version of this kit followed z-container's "git IS the disk" advice and put the Doppler PT in the committed `.env`. That conflicted with code review (reviewers flag any committed secret as P0) and with GitHub's push protection (which can auto-revoke a `ghp_*` if it's committed). The fix: put the PT in `/home/user_skills/` (per-user, outside the project repo — the same storage class z-container uses for its account default) — git-invisible by construction, no `.gitignore` dance needed.

## Validation history (8 rounds)

- R1: cold-test + audit (17 issues) + e2e live GH Actions deploy (✅ first try, 15s, all 8 secrets pulled from Doppler).
- R2: verified audit's 2 critical API claims; iterated SKILL.md applying all 17 fixes.
- R3: fresh peer-review subagent — 17/17 fixes verified FIXED. Applied 3 polish items.
- R4: examined a parallel workstream's repo; integrated clearly-better findings (`expire_at`, POST echo warning, two-tier model).
- R5: ruthless line-by-line audit with "response is in LLM context anyway" lens (339 → 240 lines). Dropped security theater (length-only verify, mktemp/chmod 600/shred for transient outputs).
- R5+: tested "do we even need SKILL.md?" with fresh sub-agent. Without SKILL.md, friction 4/5 (3 of those points were an environment-specific bash tool redaction quirk). With minimal SKILL.md, friction 1/5. Cut SKILL.md 240 → 74 lines.
- R5++: polished — dropped magic word, simplified handover labels, added bash-redaction as fact #5.
- R5+++: dropped the mint-default reflex (theater for chat session — PT is already burned, ST doesn't reduce blast radius); reclassified "Beyond the basics" into situational specifics.
- **R5++++**: split into SKILL.md (always loaded) + DEPLOY.md (lazy-loaded for deployment tasks). 3-wave usability test (simple vault → dev scenario → deployment) — friction 1/5, 2/5, 2/5. PT verified intact at the end of each wave (the 🚨 "don't revoke the master PT" rule held). Added z-container-kit pairing note + fixed the .env conflict (fact #4 now says the PT goes in the committed `.env` per z-container law 9; secrets fetched from Doppler for runtime use are not committed).
- **R6 (supersedes R5++++'s fact #4 change)**: pivoted PT persistence to `/home/user_skills/${ZK_PREFIX}-doppler.env` (mode 0600) — outside the project repo, so git/scanners/reviewers/GitHub push protection can't see it. This REVERSES the R5++++ decision to put the PT in the committed `.env` (that posture conflicted with code review and push protection; see "The tension we resolved" above). Added the cat-leak warning.
- **v2 (onboarding stress-test)**: 19 audit fixes (F1-F9 + m1/m9/m13/M3/M4/M7), incl. the config-naming convention (dev/stg/prd + `dev_personal` handling) and Path C (vault-sourced GitHub bootstrap).
- **OF rounds**: OF-1 critical — redactor extended beyond `ghp_*` (dp.pt/dp.st/cfat/sbp patterns); OF-4 Path C; OF-11 `dev_personal` convention expanded.
- **Rounds 12-13**: re-exports synced from the work repo (realistic-length redactor self-test fakes, fake token lengths).
- **v2.7.2 / v2.7.3**: fresh sub-agent review rounds 1-2, all fixes applied.
- **fix #3**: fact #4 no longer contradicts fact #5.
- **v2.7.4**: local-first sources — read the local `/home/user_skills/` copies of both kits before any raw.githubusercontent.com fetch (owner feedback); "rare ops" retitled to "special-case operations" (trigger-based framing).

## How this differs from `secrets-vault-agent-kit`

The parallel workstream (slightly newer model) ships 6 files + 3 helper scripts. This kit is 2 docs (SKILL.md + SKILL-DEPLOY.md) + 1 script (scripts/zdoppler-smoke, since v2.10). Same conclusion by independent paths.

Where the two kits contradict on technical findings, this kit kept its own verified findings (per user instruction): the per-secret `/secret?name=X` query endpoint (parallel kit tested only the path-style `/secrets/{NAME}` that 404s); the simpler `access:"read"` mint body field; the fact #2 "don't mint reflexively" rule.

## License

MIT. Use freely.

## v2.10.2: knowledge-first alignment

Path C (vault-sourced bootstrap) tail now carries a tool-neutral
annotation (hand-written one-line config + z-container's minimal save path)
and a ZK_PREFIX-at-step-1 note, so a no-kit-tools agent can run the whole
flow. Confirmed knowledge-first by line-by-line sub-agent audit (the 5
facts and the smoke fallback were already tool-neutral).

## v2.10.1: SKILL.md fat-cut

Same treatment z-container-kit v5.2.0 got: SKILL.md inventorized line by
line (V01–V27), history/version narratives and provenance tags killed,
cross-kit re-teaching compressed to a 12-line pairing header, every recipe
kept verbatim. 257 → 236 lines, zero information loss (sub-agent audited
both directions; the smoke-tool exit-code claim corrected to match the
script's actual empty-config behavior: exit 0 + warning).

## v2.9.0: multi-track-safe PT storage (atomic + fresher-wins)

z-container-kit v5 established the static rule: `/home/user_skills` is
shared across all concurrent chats, has no git (no merge/rebase/conflict
handling), and is read-only for sessions except sanctioned zero-collision
credential placements. This kit's `${ZK_PREFIX}-doppler.env` is one of
them, and v2.9 makes the fact #4 write recipe comply:

- **atomic** — same-dir tmp + `mv` (a parallel session reading the file sees
  old-or-new, never partial bytes); the old bare `cat >` redirect is gone;
- **fresher-wins** — a paste only overwrites when the stored
  `DOPPLER_PT_STORED_AT` is not newer; a stale PT from an older chat can
  never clobber a fresh rotation a parallel session already wrote;
- the pairing header places this kit inside z-container's static rule
  (the `${ZK_PREFIX}-doppler.env` write is a sanctioned placement).

Seeds policy (closing UF-1, decided): agents never write a `dp.pt.*` seed
into any Doppler config — the PT arrives via handover, lives only in this
file, and the user rotates it after the session.

## v2.8.0: v4-zero-install compose + self-refresh

z-container-kit v4.0 removed per-project kit copies: helpers run from the
canonical per-account package (`/home/user_skills/z-container-kit/scripts/`)
and read project identity from `.agents/config`. This kit's smoke-test /
bootstrap paths now use those canonical paths. Also: this kit's installed
copy at `/home/user_skills/secrets-vault-kit/` is refreshed
copy-then-swap (atomic; a parallel session never sees a half-written kit):

```bash
SRC=/tmp/my-project/svk-clone   # an updated clone of this repo
DST=/home/user_skills/secrets-vault-kit
IN="$DST.incoming.$$"; OLD="$DST.old.$$"
rm -rf "$IN" && cp -r "$SRC" "$IN" && rm -rf "$IN/.git"
[ ! -d "$DST" ] || mv -f "$DST" "$OLD"   # first install: nothing to move aside
mv -f "$IN" "$DST" && rm -rf "$OLD"
```

(Rename-aside swap — v2.9: the canonical dir is renamed AWAY and the staging
 tree renamed IN, two atomic renames; a parallel session reading the kit
 sees the old or the new tree, never a mix, and the live copy is never
 `rm -rf`'d before its replacement exists — matching what the kit's own
 SKILL.md preaches. The per-run `$$` staging name means concurrent refreshes
 can't clobber each other. Note the platform consumes
 `/home/user_skills/*.zip` at sub-agent spawn; if you keep a zip for delivery,
 rebuild it after refreshing.)

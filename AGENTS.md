# AGENTS.md — Zantor

AI-powered auditable Android terminal. Positioning: "the AI terminal you can audit."

## Repo layout
- `THREAT_MODEL.md` — agent-loop threat model (canonical; update on any architecture change)
- `docs/PLAN.md` — approved milestone plan (M0–M6)
- `docs/SPIKE-PREFIX.md` — custom-prefix build spike (verdict: provisional GO on option (b))
- `docs/NAME-AVAILABILITY.md` — name research; **do not ship as "Terminus"** (blocked by Tabby/Terminus)
- `docs/PATCHES-termux-packages.diff` — our patches over upstream termux-packages
- `termux-packages/` — git clone of upstream, local patches applied (see diff file)
- `termux-app/` — git clone of upstream termux-app (GPLv3); fork base for the app

## Decisions (locked)
- GPLv3 app (fork of termux-app); monetization only via proprietary hosted backend
- Custom app ID/prefix `com.zantor.terminal` (placeholder; final name TBD, Kersos is clear)
- Hybrid AI: cloud key for agents; on-device model limited to explain/autocomplete/risk-flag
- Dual distribution: Play-limited + F-Droid/GitHub full build
- iOS out of v1
- Package-prefix gate: if full bootstrap build exceeds ~2–3 days or core packages break,
  fall back to keeping Termux app ID for beta

## Build environment
- App SDK: `~/Android/Sdk` (build-tools, cmake, ndk, platforms, cmdline-tools)
- Package builds: Docker image `ghcr.io/termux/package-builder`, container
  `zantor-builder`, **must run `--privileged`** (fuse/overlayfs needs it; host has
  no passwordless sudo so `run-docker.sh`'s AppArmor step is bypassed).
  Repo mounted at `/home/builder/termux-packages`; `output/` is a docker volume
  (project dir is NTFS — `:` in versioned .deb filenames cannot be written there).
- Reachability quirk: `download.savannah.gnu.org` and `ftp.gnu.org` time out from
  this network; use archive.ubuntu.com orig tarballs as mirrors.
- Background bootstrap build log: `docker exec zantor-builder tail /home/builder/bootstrap-build-zantor.log`

## Conventions
- Threat-model controls (C1–C5) are normative; code must implement them, not weaken them
  silently. Narrow-untrusted gating, rule-based classifier, hash-chained + Keystore-anchored
  audit log.
- Upstream patches go in `docs/PATCHES-termux-packages.diff` (regenerate via
  `git -C termux-packages diff > docs/PATCHES-termux-packages.diff`).

## Status
- [x] THREAT_MODEL.md
- [x] docs/PLAN.md (approved)
- [x] M0 spike: prefix build pipeline validated (bash + dep chain)
- [x] M0: bootstrap build for com.zantor.terminal, aarch64 (2026-10-10; arm/i686/x86_64 pending; container name is `terminus-builder`)
- [ ] M0: bootstrap builds for arm/i686/x86_64
- [ ] M0: hands-on repro of kill/typing bugs (Pixel 10 Pro XL / Android 17: phantom-process kill NOT repro'd — 40/40 bg procs alive; Samsung/Xiaomi still pending; Play Protect blocks the debug install — disable `verifier_verify_adb_installs` or tap through)
- [ ] M0: competitive validation doc (needs device installs)
- [ ] M0: reserve GitHub org + domain for final name (user action)
- [x] M1: termux-app fork build with new package ID — built, installed on Pixel 10 Pro XL, verified (bash 5.3.20, apt, 577 bins, prefix baked)

<!-- agent-sync:start (managed by agent-sync - edit outside these markers) -->
## Multi-machine sync protocol (Windows + Kali)
This repo is worked on from more than one machine. **GitHub is the source of truth.**

**Before any work**
1. Run `agent-sync pull` (or `git pull --rebase`). Never start on a stale tree.
2. If `git status` shows changes you didn't make, stop and ask before touching them.
3. Read `docs/HANDOFF.md` first - it has the current task, decisions and next steps. Don't re-explore the repo if it already answers your question.

**While working (keep token use low)**
- Skip `node_modules`, `dist`, `build`, `.venv`, lockfiles and generated files unless the task needs them.
- Catch up with `git log --oneline -10` and `git diff --stat`, not full-file reads.
- Search (`rg`/`grep`) before opening files; open only the parts you need.

**When a task or loop is finished**
1. Update `docs/HANDOFF.md` (40 lines max): what changed, current state, next steps, open questions.
2. Run quick tests/lint if they exist.
3. Commit with a clear message, then run `agent-sync push` (or `git pull --rebase && git push`).
4. Never commit secrets (`.env`, keys, tokens) and never force-push. If the push fails or the rebase conflicts, stop and report.
<!-- agent-sync:end -->

# Execution Plan (approved v4)

Auditable, AI-powered Android terminal. Positioning: **"the AI terminal you can audit."**

## Pre-decisions (locked)

- **License: GPLv3** (fork of termux-app). Monetization only via proprietary hosted cloud-agent backend.
- **AI: hybrid** — cloud (user API key) for agents/complex work; on-device small model scoped to explain/autocomplete/risk-flag only (NNAPI deprecated; CPU/GPU via llama.cpp/LiteRT).
- **Distribution: dual track** — Play-limited build + F-Droid/GitHub full build (pentest toolset only in full build; self-modifying-code policy).
- **iOS: out of v1** — separate lite app later.
- **Kali honesty**: unrooted = no raw sockets / monitor mode / aircrack-class tools. Chroot ≠ kernel flashing. Product copy says this.

## M0 — Blocking pre-work (1 wk)

1. **Package-prefix spike with go/no-go gate.** Try option (b): own app ID + bootstrap rebuilt via termux-packages infra.
   - **Gate: if bootstrap build takes >2–3 days or breaks on core packages → fall back to option (a) (keep Termux app ID) for the beta; revisit after v1.**
   - Rationale: (b) makes us permanent owner of security updates for every rebuilt package.
2. **Hands-on repro**: build stock Termux on 2–3 devices (must include a Samsung or Xiaomi); reproduce phantom-proc kills and typing pain.
3. **Name/namespace**: availability checks (F-Droid, Play, GitHub, USPTO) on shortlist (Terminus, Halcyon, Axis, Nullpoint). Reserve GitHub org + domain immediately regardless of final name.
4. **Validation**: hands-on comparison vs. Termux + Claude Code/Gemini CLI, NetHunter, Warp — record what they do well/badly in `docs/COMPETITIVE.md`.

## M1 (1 wk)

Foreground service + declared `foregroundServiceType` (A14+); 16 KB page-size compliance for native libs (A15). Mostly inherited from Termux.

## M2a (4 wks) — reliability

- Kill diagnosis via `ApplicationExitInfo` → "killed by system: here's the fix for your phone."
- Per-OEM battery wizard (dontkillmyapp.com data, deep links).
- Phantom-proc killer fix via self-ADB (Wireless Debugging pairing + bundled ADB client) — **goes last, allowed to slip, must not block beta.** A14+ dev-option deep link as cheap fallback.
- Session restore spec: scrollback/cwd/history only; running processes are NOT revived — UI states this.

## M2b (4–5 wks) — input → public beta

Native input bar (cursor/selection/IME/voice), extra-keys row, command blocks + OSC 133 shell integration, fish-style history chips.
→ Public beta via r/termux.
**Success metric defined up front: kills-per-user-per-week vs. stock Termux.**

## M3 (4 wks) — AI + audit

AI bar (cloud key only, confirm-before-execute, risk-tier annotation); rule-based risk classifier (THREAT_MODEL §5/C3); hash-chained audit log with Keystore-anchored signed checkpoints and export.

## M4 (3–4 wks) — agent + MCP

Agent loop under THREAT_MODEL controls (narrow-untrusted gating, restricted workdir, immutable rules, step/token caps); MCP client (HTTP/SSE only, user-allowlisted, per-tool approval).

## M5 — extended platform

Kali proot image (full build only), root/chroot mode, on-device small model (explain/autocomplete/risk-flag).

## M6 — distribution

Play-limited vs. full flavor split finalized; store listings; beta → release.

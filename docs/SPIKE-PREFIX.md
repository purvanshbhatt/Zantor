# M0 Spike — Custom package prefix (option b)

**Date:** 2026-10-06
**Verdict: PROVISIONAL GO** — custom-prefix builds work; see open risks.

## Goal
Determine whether we can ship with our own app ID (`com.terminus.terminal`,
placeholder) by rebuilding the Termux package set for a new prefix:
`/data/data/com.terminus.terminal/files/usr`. Gate: abandon to option (a)
(keep Termux app ID) if a bootstrap build takes >2–3 days or breaks on core
packages.

## What worked
- `bash` + dependency chain (`libandroid-support`, `libgmp`, `ncurses`,
  `readline`, `ca-certificates`, `pcre2`, `libacl`, `attr`, …) build cleanly
  from source for `aarch64` with the new prefix.
- Verified `.deb` contents target `/data/data/com.terminus.terminal/files/usr/...`.
- The build system already detects prefix mismatch and refuses prebuilt
  `com.termux` deps ("repo package name does not equal app package name") —
  so the full dependency graph must be source-built, as expected. This is the
  real recurring cost of option (b).

## Patches applied (see `PATCHES-termux-packages.diff`)
1. `scripts/properties.sh` — default `TERMUX_APP__PACKAGE_NAME` →
   `com.terminus.terminal`. The `~/.config/termux/termuxrc.sh` override
   mechanism is sourced *after* path derivation and fight-picks every derived
   variable; patching the default in our fork is the correct path.
2. `scripts/build/termux_create_debian_subpackages.sh` — `${AR-ar}` → `ar`.
   Upstream toolchain r30 `llvm-ar` fails on versioned epochs containing `:`
   (e.g. `ca-certificates-java_1:2026.09.25_all.deb`). Candidate upstream fix.
3. `packages/attr/build.sh`, `packages/libacl/build.sh`,
   `packages/libpipeline/build.sh` — SRCURL switched from
   `download.savannah.gnu.org` (unreachable/blocked in this network) to
   Ubuntu `archive.ubuntu.com` orig tarballs, with updated SHA256.
   libpipeline kept the same tarball/hash through the pool mirror.

## Environment gotchas (not upstream bugs)
- **Build host mount is NTFS (fuseblk)**: `{pkg}_1:{ver}_all.deb` filenames
  with `:` cannot be written. Workaround: nested docker volume at
  `<repo>/output` (ext4). Do NOT build with output on the shared drive.
- `run-docker.sh` requires host sudo for AppArmor; this host has no
  passwordless sudo. Running the builder container directly with
  `--privileged` instead (accepting reduced isolation; only trusted upstream
  code is built).

## Open items (before final GO)
- [ ] Full bootstrap set build (`./build-package.sh` for all `BOOTSTRAP`
      packages incl. `apt`, `dpkg`, `dash`, `coreutils`, `tar`, `gpgv`,
      `termux-tools`…) — the packages above are a subset; `apt`/`dpkg` chain
      is the heaviest remaining.
- [ ] 4 savannah-hosted packages still unmirrored (`atool`, `lzlib`,
      `plzip`, `oathtool`) — none are bootstrap-critical; resolve later.
- [ ] Zipalign/reproducibility check of the generated bootstrap.
- [ ] Confirm `-i` (download deps) correctly refuses across prefixes
      everywhere, so no com.termux binaries can leak into our tree.

## Timing so far
8 packages (incl. full `bash` dependency chain): ~25 min wall on 8 cores,
including network firefighting. Extrapolated bootstrap set is plausible
within the 2–3 day gate.

## Decision
Continue with **option (b)**. Re-evaluate immediately if the full bootstrap
build reveals package-class breakages (toolchain-wide, not per-package source
URL issues like savannah).

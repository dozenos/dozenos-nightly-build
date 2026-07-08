# dozenos-nightly-build

This repository **builds AND stores** nightly DozenOS ISOs. It deliberately
diverges from upstream VyOS's own `vyos-nightly-build`, which is
**store-only** (something else builds; that repo only holds the resulting
releases). Here, [`.github/workflows/nightly.yml`](.github/workflows/nightly.yml)
does both: every night it rebuilds every DozenOS package from source, builds
a signed `generic` ISO from that fresh package set, and publishes the result
as a GitHub Release in this same repo.

See `dozenos-rebrand/DISTRIBUTION.md` (in the `dozenos/dozenos-rebrand` repo)
for the full artifact-distribution model this workflow implements, and
`dozenos-rebrand/ISO-BUILD.md` / `dozenos-rebrand/SB-SIGNING.md` for the
ephemeral apt repo and Secure Boot signing mechanisms it reuses.

## What's here

| Path | Purpose |
|---|---|
| `.github/workflows/nightly.yml` | The nightly build+sign+publish workflow. |
| `version.json` | Machine-readable pointer to the latest published nightly (kept up to date on the default branch by the workflow itself). |
| `minisign.pub` | Public minisign key used to verify a downloaded ISO's `.minisig` signature. **Public material, not a secret** — see "Verifying a download" below. |

This repo is deliberately thin: the actual build/sign/publish logic (the
ephemeral apt repo builder, the MOK cert injector, the `version.json`
generator, the sign-and-release script) lives as reusable helpers in
`dozenos/dozenos-rebrand`'s `release/` directory and is checked out and
invoked by the workflow at CI time, not copied in here.

## Releases

Each nightly Release is tagged `YYYY.MM.DD-HHMM-rolling` and carries four
assets:

| Asset | Purpose |
|---|---|
| `dozenos-<version>-generic-amd64.iso` | The installable image. |
| `dozenos-<version>-generic-amd64.iso.minisig` | Detached minisign signature over the ISO. |
| `version.json` | Frozen snapshot of that release's metadata (version, ISO name, sha256, URLs). |

`minisign.pub` (this repo's public verify key) is **not** a per-release
asset — it's a single, stable file committed here once and reused across
every release.

## Verifying a download

After downloading an ISO and its `.minisig` from a Release (or via the
`version.json` "latest" pointer committed to this repo's default branch):

1. Get the public verify key, `minisign.pub`, from this repo.
2. Check the sha256 against `version.json`'s `iso.sha256`:
   ```sh
   sha256sum dozenos-<version>-generic-amd64.iso
   # compare to version.json's iso.sha256 field
   ```
3. Verify the minisign signature:
   ```sh
   minisign -Vm dozenos-<version>-generic-amd64.iso -p minisign.pub
   ```
   A non-zero exit means: do not install this image.

Full spec: `dozenos-rebrand/DISTRIBUTION.md` §5 ("Consumer verify flow").

## Status

**Not yet live.** This repo is authored but not pushed to
`github.com/dozenos/dozenos-nightly-build`, and the `schedule` trigger in
`nightly.yml` does not fire until:

- the repo is actually created and pushed (blocked on item #20 in the
  DozenOS CI/CD plan),
- the required org secrets (see `dozenos-rebrand/CI-SECRETS.md`) are
  confirmed present and scoped to this repo, and
- the pipeline has been reviewed end-to-end.

`minisign.pub` in this checkout is currently a **placeholder** (see that
file) — a maintainer must replace it with the real public key paired with
the `MINISIGN_SECRET_KEY` org secret before the first real release.

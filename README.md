# dozenos-nightly-build

This repository **builds AND stores** nightly DozenOS images. It deliberately
diverges from the upstream project's nightly-release repository, which is
**store-only** (something else builds; that repo only holds the resulting
releases). Here, [`.github/workflows/nightly.yml`](.github/workflows/nightly.yml)
does both: whenever any DozenOS source mirror changes (plus a weekly forced
heartbeat), it rebuilds every DozenOS package from source, builds every
configured image flavor from that fresh package set, and publishes the result
as a GitHub Release in this same repo.

**Downloads:** the easiest way to get an image is the
[DozenOS nightly builds page](https://dozenos.github.io/dozenos-nightly-build/)
(rendered from this repo's `docs/`, always showing the current release).

See `dozenos-rebrand/DISTRIBUTION.md` (in the `dozenos/dozenos-rebrand` repo)
for the full artifact-distribution model this workflow implements, and
`dozenos-rebrand/ISO-BUILD.md` / `dozenos-rebrand/SB-SIGNING.md` for the
ephemeral apt repo and Secure Boot signing mechanisms it reuses.

## What's here

| Path | Purpose |
|---|---|
| `.github/workflows/nightly.yml` | The nightly build+sign+publish workflow. |
| `docs/` | The GitHub Pages download site (client-side render of this repo's Releases). |
| `version.json` | Machine-readable pointer to the latest published nightly (kept up to date on the default branch by the workflow itself). |
| `mirror-state.json` | Change-gate baseline: the mirror SHAs the last published nightly was built from. |
| `minisign.pub` | Public minisign key used to verify a downloaded artifact's `.minisig` signature. **Public material, not a secret** — see "Verifying a download" below. |

This repo is deliberately thin: the actual build/sign/publish logic (the
ephemeral apt repo builder, the MOK cert injector, the `version.json`
generator, the sign-and-release script) lives as reusable helpers in
`dozenos/dozenos-rebrand`'s `release/` directory and is checked out and
invoked by the workflow at CI time, not copied in here.

## Releases

Each nightly Release is tagged `YYYY.MM.DD-HHMM-rolling` and carries one
image file per flavor/format declared in the flavor tomls (currently the
`generic` ISO, the `kvm` ISO and its `qcow2`), a detached `.minisig`
signature next to every image, and a frozen `version.json` snapshot of that
release's metadata (version, artifact names, sha256, URLs).

`minisign.pub` (this repo's public verify key) is **not** a per-release
asset — it's a single, stable file committed here once and reused across
every release.

## Verifying a download

After downloading an image and its `.minisig` from a Release (or via the
`version.json` "latest" pointer committed to this repo's default branch):

1. Get the public verify key, `minisign.pub`, from this repo.
2. Check the sha256 against the matching entry in `version.json`'s
   `artifacts` list:
   ```sh
   sha256sum dozenos-<version>-<flavor>-amd64.iso
   # compare to that entry's sha256 field in version.json
   ```
3. Verify the minisign signature:
   ```sh
   minisign -Vm dozenos-<version>-<flavor>-amd64.iso -p minisign.pub
   ```
   A non-zero exit means: do not install this image.

Full spec: `dozenos-rebrand/DISTRIBUTION.md` §5 ("Consumer verify flow").

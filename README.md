# linux-release-kit

Shared CI/CD for Linux desktop apps that are **not** Tauri (those have
[tauri-release-kit](../tauri-release-kit)): iced, COSMIC, egui, GTK-rs, C,
Go — anything that compiles to one ELF binary plus a `.desktop` file and an
icon. One reusable release pipeline builds `.deb` + `.rpm` + AppImage +
Arch `.pkg.tar.zst` natively on x86_64 and aarch64 against a pinned glibc
baseline, signs the AppImage, the RPM, the Arch package and a `SHA256SUMS`
manifest with one GPG key, verifies the draft, and publishes it — plus
companion workflows that push the same release onward to **Flathub**, a
**self-hosted pacman repository**, the **AUR**, **Fedora COPR** and the
**Snap Store**, every one of them a repack (or direct re-serve) of the
artifact users already downloaded, never a second build.

Every optional piece degrades gracefully: no credentials means that channel
skips itself with a warning, never a failed release. A *half*-configured one
fails immediately, because that is someone's intent going silently unmet.

The build step is yours (`build_command`). Packaging, signing, verification
and publishing are the kit's.

## What consumers call

| Workflow | Purpose |
| --- | --- |
| `.github/workflows/release.yml` | Tag-triggered release: build → package → sign → checksum → verify → publish |
| `.github/workflows/flatpak.yml` | Repacks the released `.deb` into a Flatpak bundle + Flathub manifest |
| `.github/workflows/arch-repo.yml` | Publishes the released `.pkg.tar.zst` into a self-hosted, Pages-served pacman repository |
| `.github/workflows/aur.yml` | Renders, validates and publishes a `-bin` PKGBUILD to the AUR |
| `.github/workflows/copr.yml` | Repacks the released `.rpm` into an SRPM and submits it to a COPR project |
| `.github/workflows/snap.yml` | Repacks the released `.deb` into a strict snap and uploads it to the Snap Store |

All six are `workflow_call` reusable workflows — fixes land here once and
every app picks them up. The last five chain off `release.yml` with `needs:`
in one caller file; see [`templates/release.yml`](templates/release.yml).

**Pinning.** Reference the kit at a reviewed commit SHA
(`…/release.yml@<sha>`), not `@main`: the workflow runs with
`contents: write` and your signing key. Bump all five references in an app
together.

### Distribution channels

| Channel | Built from | Wired by | Signed by |
| --- | --- | --- | --- |
| Direct download (`.deb` / `.rpm` / AppImage / `.pkg.tar.zst`) | source, on ubuntu-22.04 | `release.yml` | you (GPG) |
| Flathub / Flatpak bundle | the released `.deb` | `flatpak.yml` | Flathub |
| Self-hosted pacman repository | the released `.pkg.tar.zst`, re-served | `arch-repo.yml` | you (GPG) |
| Arch User Repository | the released `.deb` | `aur.yml` | — (sha256 pins) |
| Fedora COPR | the released `.rpm` | `copr.yml` | COPR |
| Snap Store | the released `.deb` | `snap.yml` | Canonical |

Not included, on purpose: a Launchpad PPA (needs full `debian/` source
packaging and vendored dependencies for reach the `.deb` + Flatpak + Snap
already cover) and a self-hosted apt/dnf repository (takes these artifacts as
input; see [docs/SIGNING.md](docs/SIGNING.md)).

### The Arch story

Arch users get three doors, in order of preference:

1. **The pacman repository** (`arch-repo.yml`) — a real repo served by
   GitHub Pages from one shared git repo covering all your apps. Users add
   three lines to `/etc/pacman.conf` once and get installs *and upgrades*
   through plain `pacman -Syu`. No AUR account, no AUR helper, and it keeps
   working when the AUR is not accepting submissions. Hosting-repo setup is
   in the [`arch-repo.yml`](.github/workflows/arch-repo.yml) header.
2. **`pacman -U <release-url>`** — the release carries a native
   `.pkg.tar.zst` (with `arch` in `formats`), so a one-off install needs no
   repo at all and still gets dependency tracking and clean uninstall.
3. **The AUR** (`aur.yml`) — the discovery listing, for when the AUR is
   open. Leave the job wired with `dry_run: true` (or without the SSH key):
   it validates every release and publishes the day you flip it on.

One version rule binds all three: Arch forbids `-` in `pkgver`, and the
hyphens are **deleted**, not underscored — `1.2.3-beta.1` becomes
`1.2.3beta.1`, which `vercmp` sorts *before* `1.2.3` (the underscore form
sorts after it, i.e. a beta would out-rank the stable release).

## The glibc rule

The one constraint that shapes this kit: glibc is forward-compatible only.
A binary built on Ubuntu 24.04 dies on 22.04 with `GLIBC_2.39 not found`,
and the AppImage bundles everything *except* glibc. So `release.yml` builds
on **`ubuntu-22.04` / `ubuntu-22.04-arm`** (glibc 2.35 — Ubuntu 22.04+,
Debian 12+, Fedora 36+, RHEL 9+) and exposes that as the `runner_x86_64` /
`runner_aarch64` inputs. Raise the baseline deliberately. Never point them at
`ubuntu-latest`.

Flatpak and Snap are exempt — they link against their own runtime — which is
why those workflows run on newer images. The AUR and COPR repacks inherit
the baseline from the `.deb`/`.rpm`.

## New app checklist

1. **Copy the caller** — `templates/release.yml` →
   `.github/workflows/release.yml` (fill in `app_display_name`,
   `product_name`; for private app repos also `releases_repo` + the
   `RELEASES_TOKEN` secret). Delete the channel jobs you do not ship.

2. **Packaging directory** — `packaging/linux/` with:
   - `nfpm.yaml` from [`templates/nfpm.yaml`](templates/nfpm.yaml) — deb +
     rpm + Arch metadata, runtime dependencies per distro, file layout
     (COSMIC apps/applets: start from
     [`templates/nfpm-cosmic.yaml`](templates/nfpm-cosmic.yaml) instead)
   - `<product_name>.desktop` from
     [`templates/myapp.desktop`](templates/myapp.desktop)
   - `<product_name>.png` — 256×256 icon whose basename equals the desktop
     file's `Icon=` key (the workflow checks)
   - `<app.id>.metainfo.xml` from
     [`templates/metainfo.xml`](templates/metainfo.xml) — needed for Flathub,
     and what GNOME Software / KDE Discover show for the native packages too
   - optional: a full hicolor icon tree (16–512 px + `scalable/`), mapped in
     `nfpm.yaml` with `type: tree`

3. **Build command.** The default is a Rust app at the repo root:
   `cargo build --release --locked --target "$RK_TARGET"`, binary at
   `target/$RK_TARGET/release/$RK_BINARY`, with fmt + clippy gates before
   the build. Anything else: `rust: false` plus your own `build_command` /
   `binary_path`. Build-time `-dev` packages go in `apt_packages`; runtime
   dependencies go in `nfpm.yaml`. These variables are exported to
   `build_command`, `binary_path` and `nfpm.yaml` alike:

   | Variable | x86_64 leg | aarch64 leg |
   | --- | --- | --- |
   | `RK_TARGET` | `x86_64-unknown-linux-gnu` | `aarch64-unknown-linux-gnu` |
   | `RK_ARCH` | `x86_64` | `aarch64` |
   | `RK_DEB_ARCH` | `amd64` | `arm64` |
   | `RK_RPM_ARCH` | `x86_64` | `aarch64` |
   | `RK_VERSION` / `RK_TAG` | `1.2.3` / `v1.2.3` | same |
   | `RK_PRODUCT` / `RK_BINARY` | `product_name` / `binary_name` | same |
   | `RK_BINARY_PATH` | resolved binary (after the build) | same |

4. **CHANGELOG.md** at the repo root. The pipeline refuses to release a tag
   `vX.Y.Z` without a `## [X.Y.Z]` heading, and that section becomes the
   release notes.

5. **Signing (optional)** — `LINUX_GPG_PRIVATE_KEY` (+ `LINUX_GPG_PASSPHRASE`)
   signs the AppImage, the RPM and `SHA256SUMS`. Full setup in
   [docs/SIGNING.md](docs/SIGNING.md). Unset = unsigned, with a notice.

6. **Channel credentials (all optional)**:
   - Flathub: none for the bundle; submitting to Flathub is a manual PR with
     the manifest the workflow emits as an artifact
   - pacman repo: `ARCH_REPO_TOKEN` (fine-grained PAT, Contents:RW on the
     hosting repo) after the one-time hosting-repo setup in the
     [`arch-repo.yml`](.github/workflows/arch-repo.yml) header
   - AUR: `AUR_SSH_PRIVATE_KEY` (public half registered on your AUR account)
   - COPR: `COPR_API_CONFIG` (the config file from
     copr.fedorainfracloud.org/api, pasted whole); create the project and
     enable its x86_64/aarch64 chroots first
   - Snap: `SNAPCRAFT_STORE_CREDENTIALS` (`snapcraft export-login`), after
     `snapcraft register <name>`; plus `snap/snapcraft.yaml` from
     [`templates/snapcraft.yaml`](templates/snapcraft.yaml)

7. **Release ritual** — the tag is the trigger. With
   [`@entro314labs/release-kit`](https://www.npmjs.com/package/@entro314labs/release-kit)
   as the local half (see below):

   ```bash
   pnpm release 0.2.0      # or: patch / minor / major / (infer from commits)
   ```

   Or by hand: bump `Cargo.toml` + `Cargo.lock`, add the CHANGELOG heading,
   commit, `git tag -a v0.2.0 -m "MyApp v0.2.0" && git push origin v0.2.0`.

8. **If a leg fails** — resume, don't recycle. The pipeline reuses an
   existing draft release for the tag, so retry via dispatch with only the
   failed leg (the other leg's assets are already on the draft):

   ```bash
   gh workflow run release.yml --ref v0.2.0 -f tag=v0.2.0 -f build_targets=linux-aarch64
   ```

   `--ref` runs the workflow file as it was at the tag; the pipeline checks the tag out
   regardless of what was dispatched, so the retried leg is built from the same commit
   the other leg was. "Re-run failed jobs" on the original run is the other retry path,
   and the only one once the release is already published.

   A release already PUBLISHED for the tag is never reused — the run fails
   loudly; bump the version instead.

## The local half: release-kit

`release.yml` starts at the pushed tag. Everything before it — choosing the
version, writing it into the manifests, rolling `## [Unreleased]` into the
heading the changelog guard checks for, tagging, pushing — is
[release-kit](https://github.com/entro314-labs/release-kit)'s job. For a
Rust app the whole configuration is a `release.config.json` at the repo
root:

```json
{
  "versionFiles": ["Cargo.toml", "Cargo.lock", "packaging/linux/*.metainfo.xml"],
  "publish": null,
  "steps": ["version", "changelog", "tag", "push"]
}
```

and `"release": "release-kit"` in `package.json` scripts (or
`npx @entro314labs/release-kit@<pinned>`). It stops at `push` on purpose: the
tag push triggers this pipeline, and the pipeline owns building, signing and
the GitHub release. Do NOT add `release` to `steps` — both would try to
create it and the second fails. `Cargo.lock` is scoped to the crate named in
the sibling `Cargo.toml`, so dependency versions in it are left alone. A
half-finished run is resumed by re-running it.

The MetaInfo file is in that list because AppStream carries its own copy of
the version, and a `<release>` entry that disagrees with the binary is what
software centres show users. `templates/metainfo.xml` marks the tag with
`<!-- x-release-kit-version-date -->`, which rewrites the version and the date
together on every release; the `<description>` stays yours to write. Needs
release-kit 2.9.0 or newer — before that the marker is inert and the entry
silently goes stale.

This kit has no post-release version-bump job — with release-kit writing
the version *before* the tag there is nothing left to bump afterwards, and
two writers racing on `Cargo.toml` is how the tauri kit grew its
`post_release_bump` switch.

## Asset naming

Every downstream workflow matches on these exact names; they are built from
`product_name` and the bare version, never from what a tool happens to emit:

| Format | x86_64 | aarch64 |
| --- | --- | --- |
| `.deb` | `<product>_<ver>_amd64.deb` | `<product>_<ver>_arm64.deb` |
| `.rpm` | `<product>-<ver>-1.x86_64.rpm` | `<product>-<ver>-1.aarch64.rpm` |
| Arch | `<product>-<pkgver>-1-x86_64.pkg.tar.zst` (+ `.sig`) | `<product>-<pkgver>-1-aarch64.pkg.tar.zst` (+ `.sig`) |
| AppImage | `<product>_<ver>_x86_64.AppImage` | `<product>_<ver>_aarch64.AppImage` |
| Checksums | `SHA256SUMS`, `SHA256SUMS.asc`, `<FPR>.asc` | |

A prerelease `1.0.0-beta.1` keeps its dash in the `.deb`/AppImage names and
uses RPM's tilde form in the `.rpm` (`1.0.0~beta.1`, which sorts *before*
`1.0.0` as it should). The Arch `pkgver` deletes the hyphens
(`1.0.0beta.1` — see "The Arch story" for why not underscores), the AUR
follows the same rule, the Snap Store channel maps to `beta`, and GitHub
marks the release as a prerelease.

## Cross-repo tokens

Needed only for private app repos (per app, or org-level shared to selected
repos):

- `RELEASES_TOKEN` — fine-grained PAT, **Contents: Read and write** on the
  releases repo only. Required whenever `releases_repo` is set. The AUR,
  COPR and Flathub recipes point at the release assets and download them
  anonymously on every user's machine — a private releases repo cannot back
  them.

Fine-grained PATs EXPIRE; calendar the renewal.

## Cost notes

Everything here runs on Linux runners; GitHub-hosted ARM runners are free on
public repos and bill at the standard Linux rate on private ones. The levers
that matter:

1. **Gates before builds** — with `rust: true` the fmt/clippy gates run
   before the compile, so a lint failure costs seconds.
2. **Ship only what you sell** — `targets` drops a leg you don't ship;
   `formats` drops a format nobody downloads.
3. **Resume instead of recycling** — a failed leg is re-dispatched alone
   against the same draft (see the release ritual).
4. **Channels are cheap** — flatpak/arch-repo/aur/copr/snap each take a few
   minutes and download rather than rebuild. Run them with `dry_run: true` /
   no credentials first; they validate end to end without publishing.

## Docs

- [docs/SIGNING.md](docs/SIGNING.md) — the GPG key, what it signs, what
  users run to verify, pacman/apt/dnf repository notes
- [templates/](templates/) — the caller workflow, `nfpm.yaml` (+ the COSMIC
  variant `nfpm-cosmic.yaml`), desktop entry, AppStream MetaInfo and
  `snapcraft.yaml` to copy into an app

## Relation to the other kits

| Kit | For | Model |
| --- | --- | --- |
| [release-kit](../release-kit) | any repo | local: version → changelog → tag → push |
| [tauri-release-kit](../tauri-release-kit) | Tauri apps (mac/win/linux, updater, stores) | reusable workflows |
| **linux-release-kit** | non-Tauri Linux desktop apps | reusable workflows |
| [go-release-kit](../go-release-kit) | Go CLIs | copy-in GoReleaser templates |

The Flatpak and AUR workflows here started as the tauri kit's and diverge
only where the Tauri assumptions did (default runtime, default dependencies,
asset lookup by exact name). Fixes to the shared logic belong in both.

## License

[MIT](LICENSE)

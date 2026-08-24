# Linux signing

One GPG key, three signatures per release, zero runtime enforcement. This
page is the setup and the reasoning; the workflow does the rest.

## Who signs what

| Artifact | Mechanism | Where it happens | Verified by |
| --- | --- | --- | --- |
| AppImage | Embedded GPG signature (`SIGN=1` via appimagetool) | `release.yml` build leg | `./App.AppImage --appimage-signature`, the AppImage `validate` tool |
| `.rpm` | Embedded GPG signature (`rpmsign --addsign`) | `release.yml` build leg | `rpm -K` after `rpm --import` |
| `.deb` | **Not signed** (convention — apt trusts repository metadata, not packages) | — | `SHA256SUMS` |
| `SHA256SUMS` | Detached armored signature `SHA256SUMS.asc` | `release.yml` checksums job | `gpg --verify` |
| Flatpak bundle / Flathub | OSTree repo-level signing | Flathub's infrastructure | Flatpak |
| Snap | Store assertions | Canonical | snapd |
| AUR | None (sha256 in the PKGBUILD pin the `.deb`) | — | makepkg |
| COPR | COPR signs the built RPMs with the project key | COPR's infrastructure | dnf |

The public key is uploaded next to the artifacts as `<FINGERPRINT>.asc` on
every signed release, so install docs can link to a URL that moves with the
release.

## What it buys you

Nothing on Linux refuses to run an unsigned binary, and AppImage never
checks its own signature at launch. Signing is therefore optional here
(the pipeline ships unsigned with a notice) and worth doing anyway: it gives
security-conscious users verifiable provenance, gives *you* tamper-evidence
over the release pipeline, and becomes mandatory the day you host your own
apt/dnf repository, where unsigned metadata produces loud warnings or
refusals. The store channels (Flathub, Snap, COPR) handle trust at the
platform level, which is what most users actually rely on.

## One-time setup

1. **Generate a key** dedicated to the project (not your personal key):

   ```sh
   gpg --full-gen-key
   # Kind: (9) ECC sign and encrypt, curve Ed25519 — or (1) RSA 4096
   # Expiry: 2y   Name: "MyApp Release Signing"   Email: releases@example.com
   gpg --list-secret-keys --keyid-format long   # note the fingerprint
   ```

   Set an expiry. It limits the damage of a silent compromise; extend it
   (`gpg --edit-key <FPR>` → `expire`) before it lapses and re-publish the
   public key. Expired keys still verify old releases.

2. **Back it up offline** — a password manager or an encrypted drive:

   ```sh
   gpg --armor --export-secret-keys <FPR> > myapp-release-signing.private.asc
   ```

   Losing it means users who imported the old key see verification failures
   on the next release, and every install doc needs updating.

3. **Repository secrets** (per app, or org-level shared to selected repos):

   ```sh
   gpg --armor --export-secret-keys <FPR> | gh secret set LINUX_GPG_PRIVATE_KEY
   gh secret set LINUX_GPG_PASSPHRASE            # REQUIRED if the key has one
   gh secret set LINUX_GPG_KEY_ID --body <FPR>   # optional; defaults to the first key
   ```

   The armored key can also be stored base64-wrapped; the workflow accepts
   both. The passphrase is not optional in CI: gpg cannot prompt, and the
   workflow's `pinentry-mode loopback` + preset-passphrase dance is what lets
   `rpmsign` (which drives gpg itself) sign non-interactively.

4. **Publish the public key** somewhere stable that is *not* the release
   (your website, the repo README) so a compromised release cannot also
   swap the key users compare against:

   ```sh
   gpg --armor --export <FPR> > MYAPP-RELEASE-KEY.asc
   ```

## Behaviour in the pipeline

- **No `LINUX_GPG_PRIVATE_KEY`** → unsigned AppImage and RPM, `SHA256SUMS`
  without `.asc`, one notice in the preflight job. Valid release.
- **Key present** → the build leg imports it, proves it can sign *before*
  compiling anything (wrong passphrase fails in seconds, not after the
  build), then:
  - exports `SIGN=1 SIGN_KEY=<FPR> APPIMAGETOOL_FORCE_SIGN=1` for
    linuxdeploy/appimagetool. `FORCE_SIGN` matters: without it a signing
    failure produces a byte-identical *unsigned* AppImage and exit 0.
  - runs `rpmsign --addsign` on the `.rpm` and verifies it with `rpm -K`
    against the same key — a mismatch is a hard error because the pipeline
    did the signing itself.
  - checks the AppImage with `--appimage-signature` (advisory: a missing
    signature here is a warning, since `FORCE_SIGN` already failed the
    build on a real signing error).
- **Checksums job** → downloads every asset on the draft, writes
  `SHA256SUMS`, signs it, verifies the signature, uploads both plus
  `<FPR>.asc`.
- **Verify job** → refuses to publish if `SHA256SUMS.asc` is missing while
  a key is configured.

## What users run

Put this in your install docs, with your fingerprint:

```sh
# Checksums + signature (covers the .deb too)
curl -LO https://github.com/ORG/REPO/releases/download/vX.Y.Z/SHA256SUMS
curl -LO https://github.com/ORG/REPO/releases/download/vX.Y.Z/SHA256SUMS.asc
gpg --keyserver keys.openpgp.org --recv-keys <FPR>   # or import MYAPP-RELEASE-KEY.asc
gpg --verify SHA256SUMS.asc SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing

# RPM — import once, then dnf/rpm verify every install
sudo rpm --import https://github.com/ORG/REPO/releases/download/vX.Y.Z/<FPR>.asc
rpm -K myapp-X.Y.Z-1.x86_64.rpm

# AppImage
./myapp_X.Y.Z_x86_64.AppImage --appimage-signature
# or, with the AppImage project's validator:
./validate-x86_64.AppImage myapp_X.Y.Z_x86_64.AppImage
```

## If you later run your own apt / dnf repository

The same key signs repository metadata — `gpg --clearsign -o InRelease
Release` and `gpg -abs -o Release.gpg Release` for apt, `repodata/repomd.xml
.asc` for dnf. Standalone `.deb` files stay unsigned even then; that is how
Debian's trust model works. Tools like `aptly`, `reprepro` or a hosted
package service take the released artifacts as input — this kit
deliberately stops at GitHub Releases.

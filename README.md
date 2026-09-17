# remixbuilder

Encrypted, generic GitHub Actions bridge for source repositories hosted on qdvps Gitea.

## What is public

This repository contains no project source. It only contains the trusted GitHub Actions launcher.

## Security model

- Source is exported with `git archive`.
- Source integrity uses SHA-256.
- The public release tag is `build-<HMAC-SHA256(signing_key, source_sha256)[0:32]>`; the raw source hash is not used as the public version.
- Source is encrypted with age before it leaves qdvps.
- Every submission uses one-time keys:
  - the source archive is encrypted to a fresh age keypair generated per build; its private half is wrapped (age-encrypted) to the long-term source wrap key and travels inside the one-time object;
  - the intermediate artifact is encrypted to a second fresh age keypair per build, private half wrapped to the long-term output wrap key;
  - the final 7z passphrase is random per build, wrapped to the output wrap key, and delivered as ciphertext through public dispatch inputs.
- qdvps persists no static age private keys. It holds only the two wrap public keys; the corresponding private keys exist solely as GitHub Secrets and are used only to unwrap per-build keys.
- The object server is TLS-only (`https://47.104.2.255:3001`). The runner pins the server through the self-signed certificate embedded in the workflow; plain HTTP is refused.
- Builds run on a Linux, macOS and Windows runner matrix; each leg encrypts its output separately and publish merges them into `linux/`, `macos/`, `windows/` directories of one release.
- One one-time object per runner OS is uploaded for every submission (three signed, expiring URLs); each matrix leg downloads exactly its own object, preserving strict single-use semantics.
- The whole dispatch request — URL, hashes, version, recipe, and the three one-time-key ciphertexts — is HMAC-signed; the runner rejects anything unsigned.
- GitHub runner decrypts the source only after signature and hash verification, and destroys one-time keys immediately after use.
- Publish job creates a 7z AES-256 archive with encrypted headers and the per-build passphrase.
- qdvps retrieves the release over `api.github.com`, verifies the asset SHA-256, decrypts it with the one-time passphrase held only until that retrieval, and checks `SHA256SUMS`. The passphrase file is destroyed on successful fetch; a second fetch of the same version is refused.
- Immediately after a fully successful retrieval, the encrypted release (assets included) and its tag are deleted from GitHub, so published ciphertext never accumulates.
- Workflow run history is pruned to the last 7 days by a daily job on qdvps; deleting a run removes its logs as well.

## Generic source contract

Any Gitea repository can use the bridge by including an executable:

```text
.bridge/build.sh
```

The script must be bash-compatible (Git Bash on the Windows leg) and runs on
all three runner OSes; detect the host with `uname -s` when needed.

The script receives:

```text
BRIDGE_OUTPUT_DIR=<directory where build outputs must be written>
```

For example:

```bash
#!/usr/bin/env bash
set -euo pipefail
mkdir -p "$BRIDGE_OUTPUT_DIR"
cp target/my-binary "$BRIDGE_OUTPUT_DIR/"
```

Submit from qdvps:

```bash
/opt/remixbridge/bridge-submit.sh \
  /var/lib/gitea/data/gitea-repositories/OWNER/REPO.git \
  main shell
```

Fetch and decrypt (the one-time passphrase is generated at submit time and consumed here):

```bash
/opt/remixbridge/bridge-fetch.sh \
  PUBLIC_VERSION \
  /tmp/output
```

## Known limitations (measured, 2026-09 reliability round)

- **No git metadata on runners.** Source arrives as a `git archive` tarball: `git describe`, `git submodule`, and any `.git`-dependent logic do not work inside `.bridge/build.sh`. Tag versions must be passed some other way (e.g. written to a file before submitting).
- **Symlinks become plain files on the Windows leg.** Linux/macOS preserve symlinks; Git Bash tar materializes them as regular copies (content intact, `test -L` false). Builds that depend on symlink semantics see a 1.8 MB-different world on Windows (measured: 3 symlinks → 3 extra regular files on a 10561-file tree).
- **Large trees are bounded by the transfer window.** A 159 MB source tar moved through the whole pipeline (upload 9 s on qdvps; runner download + decrypt + 10561-file extraction succeeded on all three OSes within the 20-minute object expiry). Treat ~hundreds of MB as the practical ceiling; repo-level artifacts beyond that risk the expiry window on slow trans-border links.
- **Cross-border release downloads can stall.** `bridge-fetch.sh` resumes (`-C -`) with up to 8 attempts; a 16 MB asset typically completes in one or two attempts.
- **Failing builds leave a pending one-time passphrase** on qdvps that is never fetched (swept after 14 days) and no release is published.

## Important boundary

GitHub runners must process plaintext source and plaintext intermediate output while building. Encryption protects transfer, storage, and published artifacts; it does not make GitHub an invisible execution environment.

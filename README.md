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
- The encrypted source URL is signed and expires in 20 minutes; successful download consumes the object.
- The whole dispatch request — URL, hashes, version, recipe, and the three one-time-key ciphertexts — is HMAC-signed; the runner rejects anything unsigned.
- GitHub runner decrypts the source only after signature and hash verification, and destroys one-time keys immediately after use.
- Publish job creates a 7z AES-256 archive with encrypted headers and the per-build passphrase.
- qdvps retrieves the release over `api.github.com`, verifies the asset SHA-256, decrypts it with the one-time passphrase held only until that retrieval, and checks `SHA256SUMS`. The passphrase file is destroyed on successful fetch; a second fetch of the same version is refused.

## Generic source contract

Any Gitea repository can use the bridge by including an executable:

```text
.bridge/build.sh
```

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

## Important boundary

GitHub runners must process plaintext source and plaintext intermediate output while building. Encryption protects transfer, storage, and published artifacts; it does not make GitHub an invisible execution environment.

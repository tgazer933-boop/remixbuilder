# remixbuilder

Encrypted, generic GitHub Actions bridge for source repositories hosted on qdvps Gitea.

## What is public

This repository contains no project source. It only contains the trusted GitHub Actions launcher.

## Security model

- Source is exported with `git archive`.
- Source integrity uses SHA-256.
- The public release tag is `build-<HMAC-SHA256(signing_key, source_sha256)[0:32]>`; the raw source hash is not used as the public version.
- Source is encrypted with age before it leaves qdvps.
- The encrypted source URL is signed and expires in 20 minutes; successful download consumes the object.
- GitHub runner decrypts the source only after signature and hash verification.
- Intermediate output artifact is age-encrypted.
- Publish job creates a 7z AES-256 archive with encrypted headers and a passphrase.
- qdvps retrieves the release over `api.github.com`, decrypts it, and checks `SHA256SUMS`.

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

Fetch and decrypt:

```bash
/opt/remixbridge/bridge-fetch.sh \
  PUBLIC_VERSION \
  /tmp/output
```

## Important boundary

GitHub runners must process plaintext source and plaintext intermediate output while building. Encryption protects transfer, storage, and published artifacts; it does not make GitHub an invisible execution environment.

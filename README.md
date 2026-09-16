# remixbuilder

Encrypted build bridge for private Gitea source repositories.

This repository intentionally contains no source code. It only hosts the trusted GitHub Actions workflow that downloads an encrypted source archive from a short-lived controller endpoint, verifies signatures and hashes, builds it, then publishes a password-encrypted artifact whose public version is keyed from the source SHA-256.

No raw source MD5/SHA-256 is published as the release version. Integrity checking uses SHA-256.

# Security model

`snp` is alpha, no SLA. This page says what the signature on a release checks and what it does not. It is a description, not a promise.

## What is signed

A release carries two signed files. `snp-keys` is the key statement: the recovery key signs it, and it names the release key, the signing key for releases. `snp-release` is the manifest: the release key signs it, and it lists the sha256 of `LICENSE`, `NOTICE`, `install.sh`, `snp-linux-amd64` and `snp-linux-arm64`. `install.sh` is the step that `snp-install.sh` runs after the checks pass; it is not the verifier.

`snp-install.sh` and `snp update` check the statement against the recovery key, the manifest against the statement's release keys and the genesis key embedded in `snp-install.sh` and `snp`, minus any key that statement revokes, and each file they install against the manifest. They stop at the first check that fails and install nothing. The skills ship inside the `snp` binary, so the same checks cover them.

## What the signature protects against

- A binary changed after it was signed, on the release host or on the way to you, by someone who has no signing key.
- A manifest signed by a key that the accepted statement revokes, or by a key that is neither that statement's release key nor the genesis key that `snp-install.sh` and `snp` carry. Revocation is per statement: a client applies only the revokes in the statement it accepted. The floor is the lowest statement sequence number (seq) a client accepts. It refuses older statements and revokes nothing itself.
- A key statement that the recovery key did not sign.
- An older key statement or release replayed to `snp update`. It stops when it cannot reach the Sinopia server. It refuses a statement below the server's floor or older than one it has seen, and a release that is not newer than the one it runs. The default server's floor is above seq 1, the one statement so far that trusts the genesis key, so `snp update` refuses it. On a server left at floor 1, it refuses that statement only after it has seen a newer one.

## What it does not protect against

- **A replaced `snp-install.sh`.** `snp-install.sh` is the verifier, and it is not signed: it is not listed in the manifest. Whoever can publish a release can replace it, and a replaced script can skip every check above. The inline paste avoids this: keep `snp-install.sh` whole in your setup script, as the README describes, and the copy you checked is the copy that runs. On a laptop, download it, read it and run that copy. `snp update` does not use `snp-install.sh`.
- **A bad release signed with the right key.** The signature says that the holder of the signing key published these files. It says nothing about whether they are free of bugs or harmless.
- **An older statement or release shown to a new install.** `snp-install.sh` keeps no state. When it can reach the Sinopia server, it raises its floors to the server's and refuses a statement below the floor. When it cannot, it uses the floors written into the script, and the written keys floor is 1. It then accepts an older statement and an older release, and an older statement can bring a revoked key back: the script trusts the genesis key it carries unless the statement revokes it, so a seq 1 statement lets a manifest signed by the genesis key pass. The same holds for any key that a later statement revokes, while an older statement still trusts it.
- **What happens outside the release.** The signature covers the files of a release. It does not cover the Sinopia server, what your agents write to it, or a machine that is already compromised.
- **A release you cannot download.** If the release host is down, nothing installs, and an installed `snp` stays where it is.

## How far the signing key reaches

The release key signs manifests and nothing else. Anyone who holds it, and can also publish a release, can ship an `snp` that clients accept once the Sinopia server names that version as the latest. `snp update` installs only the version the server names, and that includes the `snp update --auto` that the SessionStart hook runs at the start of a session. `snp-install.sh` does the same when it can reach the server. When it cannot, or when the server names no version, it takes the newest release on the release host. That `snp` runs as your user, so it can read what your user can read, including `SNP_TOKEN` when it is set. Turn the hook off, as the README describes, if you want to choose when `snp` changes.

The release key cannot sign a key statement. The recovery key signs statements and nothing else, and it is kept offline. A statement it signs can rotate the release key: the new statement names a new key and revokes the old one, and the next release ships it. Rotation does not change what was already installed. A compromised recovery key cannot be rotated by a statement: it needs a new `snp-install.sh` and a new `snp`.

## Key fingerprints

Compare these with the key that signs a release: the `release` line of the `snp-keys` file on that release names it.

- Recovery key: `1U/IvEldYgf2P1yKx2DlX+tuvR4fHsQmwqgV++Ynoyw=` (key id `7c39f0ce`)
- Release key: `+7fYsR5M1dgNVlOwaqhExzwaVcVpBABFUI++DAh7S6I=` (key id `40f57e32`)
- Genesis key, revoked: `581BCDWgLCK2876qn/HCFHB1pI3AtvezbTdO+PzR1Ic=` (key id `cfaf473a`)

Each is the base64 Ed25519 public key; the key id is the first 8 hex digits of its sha256. `snp-install.sh` and `snp` carry the recovery key and the genesis key. The genesis key was the first release key: the first statement named it. The current statement refuses a manifest signed by the genesis key because it revokes that key. A newer statement refuses it only while it still revokes that key, and an older statement can bring it back, as the list above says.

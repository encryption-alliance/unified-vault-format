# Implementation Notes

This document collects the checks and constraints a conforming client MUST implement beyond the pure encoding rules of the format documents. The format documents define how data is laid out and encrypted; this document defines what a client verifies before trusting that data and what it must observe when writing. The [Threat Model and Security Considerations](security-considerations.md) explain which attacks each rule prevents.

All rules in this document are normative. They are independent of the configured formats; a format specification may carry its own *Implementation Notes* section with format-specific rules that apply in addition to this document (see e.g. [AES-SIV-512-B64URL](file%20name%20encryption/AES-SIV-512-B64URL.md#implementation-notes)).

## Vault Metadata

### Validation on Unlock

Every client MUST validate `vault.uvf` on every unlock, before deriving any key from its content, and MUST reject the file if any check fails. Validation consists of the checks listed below in addition to the JWE integrity check and the format checks stated in [Encrypted Content](vault%20metadata/README.md#encrypted-content).

* The protected header contains exactly the parameters and values listed in [JOSE Header](vault%20metadata/README.md#jose-header), in particular `zip` is absent and `uvf.spec.version` is supported.
* Every element of the `recipients` array has a unique `kid`, and the set of these `kid` values equals the set of keys of `recipientDetails` in the payload.
* Every entry of `recipientDetails` contains a `jwk` if and only if the corresponding recipient's `alg` wraps the CEK with a public key.
* `seeds` is not empty, every seed is 32 bytes long, and every ID equals the first 4 bytes of `kdf(seed, 32, "seedDigest")` for its seed (see [Seed IDs](vault%20metadata/README.md#seed-ids)).
* `initialSeed` and `latestSeed` reference IDs present in `seeds`.
* `kdfSalt` is 32 bytes long.
* The [pinning](#seed-pinning) check below passes.

### Seed Pinning

`vault.uvf` carries no version counter. A [removed recipient](vault%20metadata/README.md#removing-recipients-and-compromised-keks) with continued write access to the storage can restore an earlier `vault.uvf`, dropping seeds added since and reverting `latestSeed`, or can author a new `vault.uvf` for any public-key recipient whose key it learned while it was a member. Remaining clients would then keep writing new data under a seed the adversary knows.

Since every security-relevant change to `vault.uvf` (key rotation, recipient removal) adds a seed, and since the full 32-byte seed digest cannot be forged for a seed the adversary does not know, the set of seed digests is a reliable freshness indicator. Clients MUST therefore:

1. persist, outside the vault, the digests `kdf(seed, 32, "seedDigest")` of all seeds they have observed in a successfully validated `vault.uvf`,
2. reject a `vault.uvf` unless, for every persisted digest, `seeds` contains a seed with that digest,
3. add the digests of newly observed seeds to the persisted set after successful validation.

Seed digests reveal nothing about the seeds or any key derived from them (see [Seed IDs](vault%20metadata/README.md#seed-ids)), so the persisted set needs no confidentiality. It MUST however be protected against modification and deletion by other parties, since an adversary who can replace or remove pins defeats pinning. It therefore MUST NOT be stored on the same storage as the vault.

> [!NOTE]
> Pinning only protects clients that have observed the newer `vault.uvf`. A client that unlocks a vault for the first time, or that has been offline since before a rotation, has no basis for rejecting an older or forged version. Applications SHOULD provide such clients with the expected seed digests out of band, e.g. as part of a vault invitation or from a key-management service. At minimum this is the digest of `initialSeed`, which never changes; the digest of the current `latestSeed` is required to also detect a rollback. See [Forged vault metadata](security-considerations.md#forged-vault-metadata).

### Writing

A client MUST only write `vault.uvf` based on a version it has read and [validated](#validation-on-unlock). Every write MUST:

1. generate a fresh random CEK and a fresh random `iv`,
2. wrap the CEK for every recipient listed in the validated `recipientDetails`, plus recipients the client adds, minus recipients the client removes,
3. use only the public keys contained in the validated `recipientDetails` for wrapping; adding a recipient means adding its `recipientDetails` entry and its `recipients` element in the same write,
4. carry over all seeds of the validated version, since `seeds` is append-only,
5. when removing a recipient, add a new seed and set it as `latestSeed` in the same write (see [Removing Recipients and Compromised KEKs](vault%20metadata/README.md#removing-recipients-and-compromised-keks)).

Since the CEK is wrapped anew, the writing client MUST hold the KEK of every symmetric recipient. A client that unlocked the vault with a different recipient's key, e.g. a recovery key, therefore MUST NOT write `vault.uvf` unless it replaces every symmetric recipient it cannot re-wrap for, e.g. by setting a new password.

When adding a seed, a client MUST draw a new seed if the derived ID already exists in `seeds` with a different value.

### Key Derivation Parameters

Parameters of password-based KEK derivation such as `p2c` or an application's Argon2 parameters are read from the unprotected header before any integrity check is possible. Tampering with them can only cause the unlock to fail, but clients SHOULD bound such parameters to avoid being driven into excessive memory or CPU consumption.

## File Content

### File Header

When decrypting, a file MUST be rejected if its spec version is not supported by the application or if its seed ID is not present in the `seeds` map of the vault metadata.

### File Body

The structural validation of the file body, in particular the EOF block and the block layout derived from the ciphertext size, is defined by the [general requirements](file%20content%20encryption/README.md#general-requirements) of file content encryption and by the decryption procedure of the format in use (e.g. [AES-256-GCM](file%20content%20encryption/AES-256-GCM.md#file-body-decryption)).

## File Names

### Decrypted Names

Decrypted names are untrusted input: another recipient of a shared vault can store arbitrary values, and the storage provider can replay existing objects into other positions. Before exposing a decrypted name to the file system, an application MUST reject it if it

* is empty,
* equals `.` or `..`,
* contains `/` or a NUL byte.

Applications MUST additionally reject names that are invalid on the local file system, e.g. containing `\` or `:` where applicable.

### Symlink Targets

Symlink targets are stored verbatim and may legitimately point outside the vault. Applications SHOULD validate symlink targets and warn the user on first use if the target points outside the vault.

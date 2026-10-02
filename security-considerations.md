# Threat Model and Security Considerations

This document describes the adversaries UVF is designed to resist, the properties it does and does not provide, and the residual risks that implementers and users must account for.
It is normative for implementers only where it uses RFC-2119 keywords; the rest is explanatory.

This document covers the properties of the overall design, independent of the configured formats. Each format specification carries its own *Security Considerations* section with format-specific details such as primitives, usage limits and leakage characteristics; these apply in addition to this document and MUST be observed for the formats in use (see [AES-256-GCM](file%20content%20encryption/AES-256-GCM.md#security-considerations) and [AES-SIV-512-B64URL](file%20name%20encryption/AES-SIV-512-B64URL.md#security-considerations)).

> [!NOTE]
> UVF encrypts each node individually and leans on the underlying file system (see [Design Requirements](Requirements.md)).
> Several of the limitations below are direct, deliberate consequences of that choice — they are the price of atomic, O(1) file-system operations and are unlikely to be "fixed" without abandoning core design goals.

## Threat Model

### Assets

UVF aims to protect the confidentiality and the integrity/authenticity of:

* **A1 — File contents and symlink targets**
* **A2 — Node names** (files, directories, symlinks).
* **A3 — Directory structure** — which node is nested where.

### Adversaries

* **T1 — Passive storage provider (honest-but-curious).** Can read every ciphertext object and every historical version retained by the sync/storage backend. Holds no vault keys.
* **T2 — Active storage provider.** In addition to T1, can write, reorder, duplicate, roll back, relocate, and delete ciphertext objects arbitrarily and at any time. Holds no vault keys. **This is the primary adversary UVF is designed against.**
* **T3 — Malicious recipient (insider).** Possesses a valid vault key and can therefore decrypt the metadata payload, derive every subkey, and produce validly-encrypted objects. Relevant to shared vaults with more than one recipient.

A network attacker with read/write access to the transport is subsumed by T1/T2; securing the transport (e.g. TLS) is the application's responsibility and out of scope.

### Trust assumptions (out of scope)

* **Key material retrieval is sound.** How each recipient's key decapsulates the JWE Content Encryption Key (the KEK workflow) is application-specific and assumed correct; see [Vault Metadata](vault%20metadata/README.md).
* **The cryptographic primitives are secure** (for the configured content encryption, name encryption and KDF), and the CSPRNG used for seeds, dirIds, keys, and nonces is cryptographically strong.
* **The client endpoint is trusted** — its memory, its random source, and its correct implementation of this spec. Endpoint compromise and side channels (timing, cache) are out of scope.
* **Availability is not guaranteed.** An active provider (T2) can always withhold, corrupt, or delete data; denial of service is out of scope.

### Security goals (against a keyless adversary, T1/T2)

* **G1** Confidentiality of file/symlink contents (A1).
* **G2** Confidentiality of node names (A2).
* **G3** Ciphertext directory's storage location can't be linked to its corresponding position in the cleartext hierarchy (A3).
* **G4** Per-object integrity: no undetected modification or [truncation](#truncation-and-the-eof-block) of the bytes of a single object.
* **G5** Name–parent binding: a node name cannot be undetectably relocated to a different directory (A3).

### Non-goals (explicitly NOT provided)

* **N1** Freshness / rollback resistance (see [Rollback and replay](#rollback-and-replay-no-freshness)).
* **N2** Binding of a regular file's *content* to its *name or location* (see [File content is not bound to its name](#file-content-is-not-bound-to-its-name-or-location)).
* **N3** Metadata hiding — exact file sizes, node counts, per-directory fan-out, name lengths, name equality over time, and access patterns/timing are all observable (see [Metadata leakage](#metadata-leakage)).
* **N4** Availability / denial-of-service resistance.
* **N5** Forward secrecy or post-compromise security for data already written (see [Key rotation](#key-rotation-does-not-re-key-existing-data)).
* **N6** Protection against a malicious recipient (T3) (see [Shared vaults](#shared-vaults-and-malicious-recipients)).

## Security Considerations

### File content is not bound to its name or location

Nothing binds a regular file's ciphertext to the name or directory it occupies. An active provider (T2) can therefore, using only the vault's own genuine ciphertexts:

* swap the content of two files within a directory (names unchanged),
* relocate a file's content into a different directory, or
* present a file where a directory previously stood.

All such objects still authenticate, and the client detects nothing.
This is a structural consequence of Requirement 2 (renaming/moving a node must not rewrite its content) combined with deterministic name encryption (a node's ciphertext name must be a function of its cleartext name and parent only, so that the file system can enforce name uniqueness in O(1)).
Binding content to location would break atomic moves; binding it to the name would break O(1) collision detection.
A keyless adversary is limited to **replaying the user's own existing objects**; it cannot forge new content.

### Node type and directory relocation

A node's type is currently inferred only from structure (a plain file; a directory containing `dir.uvf`; a directory containing `symlink.uvf`), and a directory's subtree is located from the `dirId` read out of its `dir.uvf`.
Nothing binds the type, or the `dirId`, to the name slot the node occupies.
An active provider (T2) can therefore relocate one directory's subtree under another name, or convert a directory into a symlink (and vice versa) by substituting a validly-encrypted metadata object harvested elsewhere.

### Rollback and replay (no freshness)

UVF carries no monotonic version counter, timestamp, or generation number inside any authenticated object.
`uvf.spec.version` versions the *specification*, not the vault contents.
An active provider (T2) that retains earlier versions can therefore, undetectably:

* roll a single file back to earlier content (same node identity → passes every check),
* roll a symlink back to an earlier target,
* restore an earlier `vault.uvf` — reverting `latestSeed` and dropping seeds added since — thereby undoing a key rotation, or
* restore an earlier self-consistent snapshot of the whole vault.

This is the fork/rollback-consistency limitation inherent to per-file, sync-oriented encryption: a version counter inside per-file authenticated data is defeated by rolling back the entire snapshot.

> [!IMPORTANT]
> Applications that require rollback resistance MUST maintain monotonic state out of band — at minimum, persisting the highest seed count / version seen and rejecting a `vault.uvf` that regresses.
> This is outside the pure file format.

### Truncation and the EOF block

A zero-byte EOF block is appended whenever the final data block is full (and the empty file consists of the EOF block alone), letting a decryptor authenticate the exact cleartext size — see the encoder and decoder rules under [General requirements](file%20content%20encryption/README.md#general-requirements).
The property holds because decryptors are **required** to enforce it: a conforming decryptor MUST reject a file whose final data block is full but is not followed by the EOF block (truncation), and MUST reject a zero-byte block in any other position.
An active provider (T2) that strips trailing blocks therefore yields a file that fails validation rather than one that silently decrypts short.

> [!NOTE]
> This detects truncation of a given file version, not [rollback](#rollback-and-replay-no-freshness) to an earlier, validly-terminated version — an active provider can still replace a file wholesale with a shorter earlier version it retained.

### Metadata leakage

Against a keyless adversary (T1/T2), UVF does not hide:

* **Exact file sizes.** The construction is length-preserving — a fixed header and a format-specific overhead.
* **Per-directory node counts.** Directory paths are one-way derivations of the `dirId`, which hides *nesting*, but the number of directories and the number of entries in each are visible.
* **Name lengths**, depending on the format's encoding.
* **Name equality over time.** Name encryption is deterministic, so within one directory a recurring ciphertext name reveals that the same cleartext name recurred (e.g. delete-then-recreate, or rename-away-and-back). Cross-directory correlation is prevented by the distinct parent `dirId`.
* **Access patterns and timing.**

Determinism is required for stable lookup, O(1) file-system collision detection, version restoration and efficient directory listing including file size.

### Key rotation does not re-key existing data

Rotation appends a seed and advances `latestSeed`.
Directory seeds are immutable and existing objects are not re-encrypted, so rotation affects only newly created directories' names and newly written content.
Every seed ever used remains in the append-only `seeds` map and stays decryptable from the current `vault.uvf`.
Rotation therefore provides **no forward secrecy and no remediation for a leaked seed** — the old seed still decrypts all data written under it.

> [!IMPORTANT]
> Recovering confidentiality after a seed or KEK compromise requires re-encrypting affected data under a fresh seed and rotating the KEK; rotation alone does not achieve this.

### Shared vaults and malicious recipients

In a multi-recipient vault the Content Encryption Key is shared: every recipient can derive every seed and thus decrypt the entire vault, and — holding valid keys — can also produce valid new ciphertext (T3).
Sharing a vault is full mutual trust.

Removing a recipient requires rotating the KEK, and (per the section above) does not retroactively protect data that the former recipient could already read.

### Cryptographic agility and quantum resistance

All currently defined content encryption, name encryption and KDF variants retain roughly 128-bit security against a quantum adversary (Grover); see the security considerations of each variant. The exposure is the JWE key-management layer:

* The metadata payload holds **all seeds** — immutable and append-only — so it is the vault's long-lived master secret. Any recipient `alg` that is broken later exposes the vault's entire history retroactively (harvest-now-decrypt-later).

> [!WARNING]
> Recipient key-wrapping algorithms based on RSA or (EC)DH are broken by a quantum adversary (Shor) and subject to harvest-now-decrypt-later against the all-seeds payload.
> Applications with long-lived confidentiality requirements SHOULD prefer post-quantum or hybrid key wrapping. Known-broken algorithms (e.g. RSA1_5) MUST NOT be used.

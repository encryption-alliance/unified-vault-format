# File Name Encryption

## Approved File Name Formats

File name formats are specified by the [vault metadata file](../vault%20metadata/README.md) in its `nameFormat` field.

This is an exhaustive list of file name formats that have been defined in this version of the specification and MUST be supported by conforming applications:

| Format ID | Description | Properties | Restrictions |
|---|---|---|---|
| [AES-SIV-512-B64URL](AES-SIV-512-B64URL.md) | encrypt using AES-SIV, then base64url-encode file name, case-sensitive | just ASCII characters in ciphertext; case-sensitive | 16 byte overhead<br>4/3 expansion |

## General requirements

All current and future formats must fulfil the following requirements:

* A format **MUST** be deterministic: the same cleartext name, parent `dirId` and seed **MUST** always yield the same ciphertext name.
  This is required for stable lookup, O(1) file-system collision detection, version restoration and efficient directory listing.
* A format **MUST** bind the ciphertext name to the parent `dirId`, so that a name cannot be moved to another directory undetected.

## Validation of Decrypted Names and Symlink Targets

Decrypted names and symlink targets are untrusted input: another recipient of a shared vault can store arbitrary values, and the storage provider can replay existing objects into other positions. Before exposing a decrypted name to the file system, an application MUST reject it if it

* is empty,
* equals `.` or `..`,
* contains `/` or a NUL byte.

Applications MUST additionally reject names that are invalid on the local file system, e.g. containing `\` or `:` where applicable.

Symlink targets are stored verbatim and may legitimately point outside the vault. Applications SHOULD validate symlink targets and warn the user on first use if the target points outside the vault.

## Possible Future Formats

> [!NOTE]
> Future versions of this standard might add further formats or deprecate existing ones. Existing formats MUST NOT be changed while keeping the same ID, though.

| Format ID | Description | Properties | Restrictions |
|---|---|---|---|
| NONE | No file name encryption, just file extensions | - | no confidentiality |
| AES-SIV-512-B32-CI | encrypt using AES-SIV, then base32-encode, apply case information on encoded ciphertext | just ASCII characters in ciphertext; case-insensitive | :warning: leaks case information <br>16 byte overhead<br>8/5 expansion |
| AES-SIV-512-B4K | encrypt using AES-SIV, then base4k-encode | short file names (using multi-byte chars); case-sensitive | 16 bytes overhead<br>unicode required |
| ... | ... | ... | ... |
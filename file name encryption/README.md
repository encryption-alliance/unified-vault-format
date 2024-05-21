# File Name Encryption

## Approved File Name Formats

File name formats are specified by the [vault metadata file](../vault%20metadata/README.md) in its `nameFormat` field.

This is an exhaustive list of file name formats that have been defined in this version of the specification and MUST be supported by conforming applications:

| Format ID | Description | Properties | Restrictions |
|---|---|---|---|
| [AES-SIV-512-B64URL](AES-SIV-512-B64URL.md) | encrypt using AES-SIV, then base64url-encode file name, case-sensitive | just ASCII characters in ciphertext; case-sensitive | 16 byte overhead<br>4/3 expansion |

## Possible Future Formats

> [!NOTE]
> Future versions of this standard might add further formats or deprecate existing ones. Existing formats MUST NOT be changed while keeping the same ID, though.

| Format ID | Description | Properties | Restrictions |
|---|---|---|---|
| NONE | No file name encryption, just file extensions | - | no confidentiality |
| AES-SIV-512-B32-CI | encrypt using AES-SIV, then base32-encode, apply case information on encoded ciphertext | just ASCII characters in ciphertext; case-insensitive | :warning: leaks case information <br>16 byte overhead<br>8/5 expansion |
| AES-SIV-512-B4K | encrypt using AES-SIV, then base4k-encode | short file names (using multi-byte chars); case-sensitive | 16 bytes overhead<br>unicode required |
| ... | ... | ... | ... |
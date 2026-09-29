# File Content Encryption

## File Header

Each file shall have a file header at offset 0 containing:

* 8 bytes general header fields:
  * 3 byte: ASCII `uvf` (big-endian) [file signature / magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)
    identifying this file as encrypted data according to this specification
  * 1 byte: uvf spec version (0-255)
  * 4 byte: ID of the seed used to derive the file key
* n bytes of _Format_-specific metadata (number of bytes depends on _Format_)
    * e.g. IVs
    * ...

## File Formats

File formats are specified by the [vault metadata file](../vault%20metadata/README.md) in its `fileFormat` field.

This is an exhaustive list of file body formats that have been defined in this version of the specification and MUST be supported by conforming applications:

| Format ID                         | Description                       |
|-----------------------------------|-----------------------------------|
| [AES-256-GCM-32k](AES-256-GCM.md#32k) | AES-GCM-encrypted blocks of 32kiB |
| ... | ... |

> [!NOTE]
> Future versions of this standard might add further formats or deprecate existing ones. Existing formats MUST NOT be changed while keeping the same ID, though.


### General requirements

All current and future formats must fulfil the following requirements:

* The *BLOCK NUMBER* (first data block is *BLOCK NUMBER* zero) **MUST** be mixed into each encrypted data block.
  This prohibits unnoticed tampering of block positions within a ciphertext file.
* A zero-byte EOF block **MUST** be appended whenever the last data block is full, i.e. whenever the cleartext length is a multiple of the block size.
  The empty file is such a multiple, so it consists of the EOF block alone.
* When decrypting, an application **MUST** validate this structure, so that the cleartext size is authenticated and can be reliably derived from the ciphertext.
  It **MUST** reject a file whose last data block is full but is not followed by the zero-byte EOF block (this indicates truncation), and it **MUST** reject a zero-byte block occurring anywhere other than as the final block following a full data block, or as the sole block of an empty file.

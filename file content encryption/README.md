# File Content Encryption

:warning: this is a working draft

## File Header

Each file shall have a file header at offset 0 containing:

* 8 bytes general header fields:
  * 3 byte: ASCII `uvf` (big-endian) [file signature / magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)
    identifying this file as encrypted data according to this specification
  * 1 byte: uvf spec version (0-255)
  * 4 byte: _Vault Key ID_
* n bytes of _Format_-specific metadata (number of bytes depends on _Format_)
    * e.g. IVs
    * ...

## File Formats

File formats are specified by the [vault metadata file](../vault%20metadata/README.md) in its `fileFormat` field.

The following file body formats have been defined:

| Format ID                         | Description                       | Required?       |
|-----------------------------------|-----------------------------------|-----------------|
| [AES-256-GCM-32k](AES-256-GCM.md#32k) | AES-GCM-encrypted blocks of 32kiB | Since version 1 |
| ... | ... | ... |


### General requirements:

All formats must fulfil the following requirements

* The *BLOCK NUMBER* (first data block is *BLOCK NUMBER* zero) **MUST** be mixed into each encrypted data block.
  This makes copying ciphertext blocks from one file to the same file at another location
  impossible.
# File Content Encryption

:warning: this is a working draft

## File Header

Each file shall have a file header at offset 0 containing:

* 8 bytes general header fields:
  * 3 byte: ASCII `uvf` [file signature / magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)
    identifying this file as encrypted data acc. to this specification
  * 1 byte: uvf spec version (0-255)
  * 4 byte: _Vault Key ID_
* _Format_-specific metadata (_Format_-specific length, specified in vault metadata)
    * e.g. file key
    * ...

Both, the _Vault Key ID_ as well as the file's _Format_ is fixed-per-vault and defined in the [vault metadata file](../vault%20metadata/README.md).

## File Formats

File formats are specified by the [vault metadata file](../vault%20metadata/README.md) in its `fileFormat` properties.

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
# File Content Encryption

:warning: this is a working draft

## File Header

Each file shall have a file header at offset 0 containing:

* [file signature / magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)
  identifying this file as encrypted data acc. to this specification
* file body offset
* file key information
* file body format (cipher mode, block size, ...)
* tbd

## File Body

General requirements:

* The *BLOCK NUMBER* (first data block is *BLOCK NUMBER* zero) **MUST** be mixed into each encrypted data block.
  This makes copying ciphertext blocks from one file to the same file at another location
  impossible.

The following file body formats have been defined:

| Format ID   | Description                       | Required?       |
|-------------|-----------------------------------|-----------------|
| AES-GCM-32k | AES-GCM-encrypted blocks of 32kiB | Since version 1 |
| ... | ... | ... |

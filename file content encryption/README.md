# File Content Encryption

:warning: this is a working draft

## File Header

Each file shall have a file header at offset 0 containing:

* plaintext file signature identifying this file to be encrypted
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
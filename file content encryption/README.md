# File Content Encryption

:warning: this is a working draft

## File Header

Each file shall have a file header at offset 0 containing:

* Random 256-bit *FILE ID*
 * This *FILE ID* **MUST** be mixed into each encrypted data block to tie that
   block to the file. This make copying ciphertext blocks from one file
   to another impossible.
* **TO BE DECIDED** should the *FILE ID* be used as a per-file encryption key
  by encrypting it with the *MASTER KEY*?

## File Body

The following file body formats have been defined:

| Format ID   | Description                       | Required?       |
|-------------|-----------------------------------|-----------------|
| AES-GCM-32k | AES-GCM-encrypted blocks of 32kiB | Since version 1 |
| ... | ... | ... |

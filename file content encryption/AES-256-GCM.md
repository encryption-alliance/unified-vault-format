# File Content Encryption using AES-256-GCM

## Format-specific file header fields
This format requires 60 additional bytes in the file header:

* 12 byte nonce
* 32 byte encrypted file content key
* 16 byte tag

The header needs to be encrypted using a 256 bit key derived from the raw key using the KDF defined in the [vault metadata file](../vault%20metadata/README.md).

```txt
headerKey := kdf(secret: vaultKey)
headerNonce := csprng(bytes: 12)
fileKey := csprng(bytes: 32)
encryptedFileKey, tag := aesGcm(cleartext: fileKey, key: headerKey, nonce: headerNonce, ad: generalHeaderFields)
header := generalHeaderFields . headerNonce . encryptedFileKey . tag
```

## File Body Encryption

The body is split up into chunks. Each chunk consists of:

* 12 byte nonce
* `n` bytes encrypted payload (see subsections)
* 16 bytes tag

```txt
cleartextBlocks[] := split(data: cleartext, maxBytes: n)
for (uint32 i = 0; i < length(cleartextBlocks); i++) {
    blockNonce := csprng(bytes: 12)
    ad := [bigEndian(i), headerNonce]
    ciphertextBlock, tag := aesGcm(cleartext: cleartextBlocks[i], key: fileKey, nonce: blockNonce, ad: ad)
    ciphertextBlocks[i] := blockNonce . ciphertextBlock . tag
}
body := join(ciphertextBlocks[])
```

### 32k

This variant uses 32740 payload bytes per block (resulting in 32768 encrypted bytes per chunk).
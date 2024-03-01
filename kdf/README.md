# Key Derivation Functions

The KDF is used to
1. derive subkeys from the vault's raw keys, which are stored in the `keys` field of the [vault metadata file](../vault%20metadata/README.md) and
2. expand these subkeys to meet the length requirements for specific algorithms.

> [!IMPORTANT]
> The KDFs listed below should be considered unsuitable for low-entropy sources and are not hard to compute. Therefore they must only be used for the aforementioned intended use case, where the input guaranteed to be a high-entropy, uniformly random secret key.

The KDF for a given vault is specified by the `kdf` field in its [vault metadata file](../vault%20metadata/README.md). This is an exhaustive list of functions that conforming applications MUST support:

| ID                | Description                       |
|-------------------|-----------------------------------|
| 1STEP-HMAC-SHA512 | Single-Step KDF as defined in [NIST SP 800-56C Rev2](https://doi.org/10.6028/NIST.SP.800-56Cr2) using HMAC-SHA512 and a salt of 128 null bytes |

> [!NOTE]
> Future versions of this standard might add further KDFs or deprecate existing ones.

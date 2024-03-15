# Key Derivation Functions

The KDF is used to
1. derive subkeys from the vault's seeds, which are stored in the `seeds` field of the [vault metadata file](../vault%20metadata/README.md),
2. expand these subkeys to meet the length requirements for specific algorithms, or
3. deterministically derive other high-entropy data such as salts or IVs

> [!WARNING]
> The KDFs listed below MAY be unsuitable for low-entropy inputs and MUST NOT be hard to compute. Therefore they must only be used for the aforementioned intended use cases, where the input is guaranteed to be a high-entropy, uniformly random secret key.

## Usage

In the context of this specification `kdf()` is a ternary function with the following three parameters:
* an input key
* an output length in bytes
* an optional context

Other parameters, such as the internally used hash function, are constant and defined during vault creation.

## Available KDFs

The KDF for a given vault is specified by the `kdf` field in its [vault metadata file](../vault%20metadata/README.md). This is an exhaustive list of functions that conforming applications MUST support:

| ID          | Description                       |
|-------------|-----------------------------------|
| HKDF-SHA512 | HKDF as defined in [RFC 5869](https://datatracker.ietf.org/doc/html/rfc5869) using SHA512, <br/> salted by the `salt` value defined in the [vault metadata file](../vault%20metadata/README.md) |

> [!NOTE]
> Future versions of this standard might add further KDFs or deprecate existing ones.
> 
> When such a future KDF doesn't support a context, it shall be appended to the salt. When it doesn't support a salt, the salt shall be appended to the input key.
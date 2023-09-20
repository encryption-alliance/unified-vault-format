# Vault Metadata

Each vault contains one vault metadata file, which holds essential information like encryption parameters.

In order to decrypt this file, a KEK is required. Retrieval of this KEK is application-specific and the workflow is not part of this spec.

## Filename and Location

The metadata file MUST be stored in the root directory of the encrypted vault structure for applications to retrieve it from remote storage without traversing deep hierarchies. Furthermore it helps the user to understand the purpose of the directory.

The file SHOULD be named `vault.uvf` (_TBD_), however application vendors may decide to use custom file extensions to register the file type with their application.

## File Format

The file contains a JWE in _JWE Compact Serialization_ format ([RFC 7516](https://datatracker.ietf.org/doc/html/rfc7516)), as it is an easy-to-implement, flexible, broadly-used and mature standard that allows to store arbitrary public metadata in its header as well as sensitive data in its ciphertext.

### Public Metadata

In order to comply with [RFC 7516, Section 4.2](https://datatracker.ietf.org/doc/html/rfc7516#section-4.2), any UVF-specific parameters MUST be prefixed with `uvf.`.

With this version of the UVF specification, the following registered header fields and values are supported:

| JOSE Header | Allowed Values | Remark |
|---|---|---|
| `alg` | `A256KW` | Further algorithms may be added in later revisions |
| `enc` | `A256GCM` | Further encryption algorithms may be added in later revisions |
| `typ` | `JWE` | |
| `cty` | `json`, ()`application/json`) | `application/` SHOULD be omitted [as per spec](https://datatracker.ietf.org/doc/html/rfc7515.html#section-4.1.10) |
| `crit` | `["uvf.spec.version"]` | |
| `uvf.spec.version` | `1` | To be increased with newer revisions of this spec |

`zip` MUST be omitted (disallowing compression of the plaintext)

### Sensitive Metadata

The JWE Ciphertext decrypts to a JSON object (as denoted by the `cty` header). Any sensitive metadata MUST be added to this.

With this version of the UVF specification, the payload contains the following data:

* `keys`: a map of key names and raw keys matching the length required for [file content encryption](../file%20content%20encryption/README.md)
* `latestKey`: referencing the key to be used for newly added data (allowing key rotation)

```json
{
    "latestKey": "QBsJ",
    "keys": {
        "HDm3": "ypeBEsobvcr6wjGzmiPcTaeG7/gUfE5yuYB3ha/uSLs=",
        "cnQp": "PiPoFgA5WUoziU9lZOGxNIu9egCI1CxKy3PurtWcAJ0=",
        "QBsJ": "Ln0sA6lQeuJl7PW1NWiFpTOTogKdJBOUmXJloaJa78Y="
    }
}
```

### Example / Test Data


```mermaid
flowchart TD
    U[fa:fa-user User] -->|enter credentials| 1(fa:fa-key Derive KEK)
    1 --> 2(Decrypt JWE)
    F[fa:fa-file vault.uvf] --> 2
    2 --> 3(fa:fa-key Use Vault Keys)
```

TODO: add example JWE and KEK
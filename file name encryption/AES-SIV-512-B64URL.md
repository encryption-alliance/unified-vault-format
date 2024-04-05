# File Name Encryption using AES-SIV-512 + Base64-URL-Encoding

## Directory Metadata

Every directory requires certain metadata that affects the file name encryption of its direct children:

* the seed used to derive keys
* the directory ID

This data is immutable and therefore linked with a directory eternally, surviving renames/moves. This data is stored in a file called `dir.uvf`, which is stored in two places:
1. Within the parent dir (except for root), where it serves a link to the child dir
2. In the child dir itself (allowing disaster recovery without the parent)

The exact file structure to store the dir ID will be discussed in more detail [below](#format-of-diruvf-and-symlinkuvf).

### Directory Seed

At the time of its creation, a directory's seed is always the `latestSeed`. In case of the root directory this happens to also be the `initialSeed`.

When navigating into a directory, the seed is read from the corresponding `dir.uvf`'s file header.

> [!NOTE]
> Child node names are encrypted with keys derived from their parent directory's seed. Due to the immutability, adding new child nodes to old parents will continue to use the old seed.
>
> Consequently, key rotation only affects child names of newly created directories.

### Directory ID

The directory ID is a unique sequence of 32 random bytes (taking the birthday paradox into account, collision probability is therefore 2^-128).

```ts
let dirId = csprng(len: 32)
```

The only exception to this is the root directory ID, which is deterministically derived from the `initialSeed` using the [KDF](../kdf/README.md):

```ts
let rootDirId = kdf(secret: initialSeed, len: 32, context: "rootDirId")
```

## Deriving Encryption Keys

All file names are encrypted using AES-SIV, which requires a 512 bit key (which is internally split into two 256 bit AES keys). Furthermore we need a 256bit key for HMAC computations. We use the directory-specific seed from `dir.uvf` and feed it into the [KDF](../kdf/README.md):

```ts
let sivKey = kdf(secret: seed, len: 64, context: "siv")
let hmacKey = kdf(secret: seed, len: 32, context: "hmac")
```

## Mapping Directory IDs to Paths

When traversing directories, the directory ID of a given subdirectory is processed in three steps to determine the storage path inside the vault:

1. Compute the HMAC of the `dirId` using SHA-256 and the `hmacKey`
1. Encoding the hash with Base32 to get a string of printable chars
1. Constructing the directory path by resolving substrings of the encoded hash relative to the `{vaultRoot}/d/`

```ts
let dirIdHash = hmacSha256(data: dirId, key: hmacKey)
let dirIdString = base32(dirIdHash)
let dirPath = vaultRoot + '/d/' + dirIdString[0..2] + '/' + dirIdString[2..32]
```

> [!NOTE]
> Due to the nature of hierarchical data structures, travering file trees is an inherently top-down process, allowing the use of one-way hash functions.

> [!TIP]
> Splitting the `dirIdString` into a path like `d/AB/CDEFGHIJKLMNOPQRSTUVWXYZ234567` is inspired by Cryptomator's former vault formats and serves two purposes:
> 1. Gather all data within a single data dir, uncluttering the root dir
> 2. Having at most `32^2` subdirectories within `d`

Regardless of the hierarchy of cleartext paths, ciphertext directories are always stored in a flattened structure. All directories will therefore effectively be siblings (or cousins, to be precise).


## Encryption of Node Names

The cleartext name of a node gets encoded using UTF-8 in [Normalization Form C](https://unicode.org/reports/tr15/#Norm*Forms) to get a unique binary representation.

The byte sequence is then encrypted using AES-SIV as defined in [RFC 5297](https://tools.ietf.org/html/rfc5297). In order to bind the node to the containing directory, preventing undetected manipulation of the folder structure, the directory ID of the parent folder is used as associated data.

Lastly, the ciphertext is encoded with unpadded base64url.

```ts
let ciphertext = aesSiv(secret: cleartextName, ad: parentDirId, key: sivKey)
let ciphertextName = base64url(data: ciphertext) + '.uvf'
```

## Ciphertext Directory Structure

### Node Types

Depending on the kind of a cleartext node, the encrypted name is then either used to create a file or a directory:

| cleartext node type | ciphertext structure                |
|---------------------|-------------------------------------|
| file                | file                                |
| directory           | directory containing `dir.uvf`      |
| symlink             | directory containing `symlink.uvf`  |

### Format of `dir.uvf` and `symlink.uvf`

Both, `dir.uvf` and `symlink.uvf` files are encrypted using the [content encryption mechanism](../file%20content%20encryption/README.md) configured for the vault.

The cleartext content of `dir.uvf` is the 32 byte dirId. The seed to encrypt this file's header is the directory seed.

The cleartext content of `symlink.uvf` is an UTF-8 string in Normalization Form C, denoting the cleartext target of the symlink.

> [!CAUTION]
> Every `*.uvf` file MUST be encrypted independently, particularly the two `dir.uvf` copies that contain the same dirId. This is required for indistinguishable ciphertexts, avoiding the leakage of the nested dir structure.

### Example Directory Structure

Thus, for a given cleartext directory structure like this...

```txt
.
├─ File.txt
├─ Symlink
├─ Subdirectory
│  └─ ...
└─ ...
```

...the corresponding ciphertext directory structure will be:

```txt
.
├─ vault.uvf
└─ d
   ├─ BZ
   │  └─ R4VZSS5PEF7TU3PMFIMON5GJRNBDWA     # Root Directory
   │     ├─ dir.uvf                         # Root Directory's dirId
   │     ├─ 5TyvCyF255sRtfrIv83ucADQ.uvf    # File.txt
   │     ├─ FHTa55bHsUfVDbEb0gTL9hZ8nho.uvf # Subdirectory
   │     │  └─ dir.uvf                      # Subdirectory's dirId
   │     └─ gLeOGMCN358UBf2Qk9cWCQl.uvf     # Symlink
   │        └─ symlink.uvf                  # Symlink's target
   ├─ FC
   │  └─ ZKZRLZUODUUYTYA4457CSBPZXB5A77     # Subdirectory
   │     ├─ dir.uvf                         # Subdirectory's dirId
   |     └─ ...                             # Subdirectory's children
   └─ ...
```

### Traversing the Example Directory Structure

#### List contents of `/`:

1. Use `initialSeed` as a seed for the root directory
1. compute `rootDirId` and corresponding ciphertext dir path -> `d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA`
1. list direct children within `d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA`
    * `dir.uvf` (file)
    * `5TyvCyF255sRtfrIv83ucADQ.uvf` (file)
    * `FHTa55bHsUfVDbEb0gTL9hZ8nho.uvf` (dir)
    * `gLeOGMCN358UBf2Qk9cWCQl.uvf` (dir)
1. For each subdirectory, determine node type
    * `FHTa55bHsUfVDbEb0gTL9hZ8nho.uvf` denotes a dir (contains `dir.uvf`)
    * `gLeOGMCN358UBf2Qk9cWCQl.uvf` denotes a symlink (contains `symlink.uvf`)
1. strip file extension and decrypt file names
    * `File.txt`
    * `Subdirectory`
    * `Symlink`

#### List contents of `/Subdirectory/`:

1. read seed and decrypt `d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA/FHTa55bHsUfVDbEb0gTL9hZ8nho.uvf/dir.uvf`
1. read `dirId` from said file and compute ciphertext path -> `d/FC/ZKZRLZUODUUYTYA4457CSBPZXB5A77`
1. Repeat dir listing procedure for `d/FC/ZKZRLZUODUUYTYA4457CSBPZXB5A77`

#### Read target of `/Symlink`:

1. decrypt file `d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA/gLeOGMCN358UBf2Qk9cWCQl.uvf/symlink.uvf`

#### Read content of `/File.txt`:

1. decrypt file `d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA/5TyvCyF255sRtfrIv83ucADQ.uvf`
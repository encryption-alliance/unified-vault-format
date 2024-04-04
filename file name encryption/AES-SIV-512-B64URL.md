# File Name Encryption using AES-SIV-512 + Base64-URL-Encoding

## Derive Required Keys

All file names are encrypted using AES-SIV, which requires a 512 bit key (which is internally split into two 256 bit AES keys). Furthermore we need a 256bit key for HMAC computations. So the first step is to look up the seed for file name encryption as denoted by the `nameKey` and pass it through the [KDF](../kdf/README.md):

```txt
sivKey := kdf(secret: nameKey, len: 64, context: "siv")
hmacKey := kdf(secret: nameKey, len: 32, context: "hmac")
```

## Directory IDs

Every directory has a unique _directory ID_, which is defined to be a sequence of 32 random bytes (taking the birthday paradox into account, collision probability is therefore 2^-128). Directory IDs are immutable and therefore linked with a directory eternally, surviving renames/moves.

```txt
dirId = csprng(len: 32)
```

The only exception to this is the root directory ID, which depends on the `nameKey`:

```txt
rootDirId := kdf(secret: nameKey, len: 32, context: "rootDirId")
```

The directory ID is stored in two places:
1. Within the parent dir (except for root), where it serves a link to the child dir
2. In the child dir itself (allowing disaster recovery without the parent)

The exact file structure to store the dir ID will be discussed in more detail below.

## Mapping Directory IDs to Paths

When traversing directories, the directory ID of a given subdirectory is processed in three steps to determine the storage path inside the vault:

1. Compute the HMAC of the `dirId` using SHA-256 and the `hmacKey`
1. Encoding the hash with Base32 to get a string of printable chars
1. Constructing the directory path by resolving substrings of the encoded hash relative to the `{vaultRoot}/d/`

```txt
dirIdHash := hmacSha256(data: dirId, key: hmacKey)
dirIdString := base32(dirIdHash)
dirPath := vaultRoot + '/d/' + dirIdString[0..2] + '/' + dirIdString[2..32]
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

```txt
ciphertext := aesSiv(secret: cleartextName, ad: parentDirId, key: sivKey)
ciphertextName := base64url(data: ciphertext) + '.uvf'
```

## Ciphertext Directory Structure

### Node Types

Depending on the kind of a cleartext node, the encrypted name is then either used to create a file or a directory:

| cleartext node type | ciphertext structure                |
|---------------------|-------------------------------------|
| file                | file                                |
| directory           | directory containing `_dir.uvf`     |
| symlink             | directory containing `_symlink.uvf` |

### Format of `_dir.uvf` and `_symlink.uvf`

Both, `_dir.uvf` and `_symlink.uvf` files are encrypted using the content encryption mechanism configured for the vault.

> [!IMPORTANT]
> The file header MUST reference the seed denoted by `nameKey`, as key rotation does not apply to file names.

The cleartext content of `_dir.uvf` is the 32 byte dirId.

The cleartext content of `_symlink.uvf` is an UTF-8 string in Normalization Form C, denoting the cleartext target of the symlink.

> [!CAUTION]
> Every `*.uvf` file MUST be encrypted independently, particularly the two `_dir.uvf` copies that contain the same dirId. This is required for indistinguishable ciphertexts, avoiding the leakage of the nested dir structure.

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
   │     ├─ _dir.uvf                        # Root Directory's dirId
   │     ├─ 5TyvCyF255sRtfrIv83ucADQ.uvf    # File.txt
   │     ├─ FHTa55bHsUfVDbEb0gTL9hZ8nho.uvf # Subdirectory
   │     │  └─ _dir.uvf                     # Subdirectory's dirId
   │     └─ gLeOGMCN358UBf2Qk9cWCQl.uvf     # Symlink
   │        └─ _symlink.uvf                 # Symlink's target
   ├─ FC
   │  └─ ZKZRLZUODUUYTYA4457CSBPZXB5A77     # Subdirectory
   │     ├─ _dir.uvf                        # Subdirectory's dirId
   |     └─ ...                             # Subdirectory's children
   └─ ...
```

# File Name Shortening

File name shortening is an **optional middleware** between the file name encryption layer and the underlying storage. It keeps every stored path component short enough for file systems and sync services that limit name lengths (typically 255 bytes), regardless of which [file name encryption](../file-name-encryption/README.md) format produced the name.

> [!NOTE]
> This layer involves no cryptography and no keys. It adds no security and serves compatibility only. It operates on already encrypted names and does not need to understand them.

## Parameters

Shortening is enabled per vault by the `nameShorteningThreshold` field in the [vault metadata file](../vault-metadata/README.md#encrypted-content). That field also sets the threshold `T`. The application passes `T` to the middleware after unlocking the vault. The middleware itself remains independent of any cryptography.

| Parameter | Value |
|---|---|
| threshold `T` | `nameShorteningThreshold` from `vault.uvf`: at least 46 (the length of a stored short name), 220 recommended (leaves headroom for suffixes added by sync services in case of conflicts) |
| hash | SHA-256, truncated to the leftmost 160 bits |
| encoding | Base32 ([RFC 4648](https://www.rfc-editor.org/rfc/rfc4648#section-6), uppercase alphabet, no padding), 32 characters |
| node suffix | `.shortened.uvf` |
| sidecar suffix | `.original.uvf` |

All lengths and hash inputs refer to the UTF-8 byte representation of a name.

## Shortening

Each path component is considered individually. A name longer than `T` is stored under a short name derived from its hash. A sibling sidecar file holds the original name:

```ts
function isLong(name: string): boolean {
  return utf8(name).length > T
}

function shortName(name: string): string {
  let hash = sha256(utf8(name))[0..20] // leftmost 20 bytes (160 bits)
  return base32(hash)
}

function storedName(name: string): string {
  return isLong(name) ? shortName(name) + '.shortened.uvf' : name
}

function sidecarName(name: string): string {
  return shortName(name) + '.original.uvf' // content: utf8(name), nothing else
}
```

The node keeps its type: a file is stored as a file named `{hash}.shortened.uvf`, and a directory as a directory of that name, whose children are mapped recursively. The sidecar is always a regular file. The middleware therefore never needs to know node types or the internals of the name format.

## Example

The upper layer creates a file whose ciphertext name is 228 bytes long:

```txt
cAp6_KasM-0d74bYyP1eOpweCA4NGgwjVfDGc6heAlXbLIVq2tMWskWpw-DGgyoC63Tjjs3weXCL3hjGfQlP_uJDocVs7kxvIKDEsnHk2GYEnbLGJgCsYcIahvxW9PLxkIXowBkEcaedqWWZNmW0Cv0e9-EaPHmiveg-uvH7Hdp0vxEbxqxRioJBDN1jmVVfV-rE8fCa6ijyBJXYem0IvMLYl4wNgi0G.uvf
```

Its truncated SHA-256 in Base32 is `WQ6QCEQX5BTXEPNQZNJLHDN52MADIVA4`. Together with a long directory name and a short one, storage looks like this:

```txt
d/BZ/R4VZSS5PEF7TU3PMFIMON5GJRNBDWA
├─ dir.uvf
├─ 5TyvCyF255sRtfrIv83ucADQ.uvf                         # short name, stored as-is
├─ WQ6QCEQX5BTXEPNQZNJLHDN52MADIVA4.shortened.uvf       # file with long name
├─ WQ6QCEQX5BTXEPNQZNJLHDN52MADIVA4.original.uvf        # → "cAp6_KasM-0d74…0G.uvf"
├─ 3LZ7XQF2N4KDPSJVW6YHCE5MAGRUBTIO.shortened.uvf       # dir with long name...
│  └─ dir.uvf                                           # ...children are unaffected
└─ 3LZ7XQF2N4KDPSJVW6YHCE5MAGRUBTIO.original.uvf        # → "…long dir name….uvf"
```

## Operations

```ts
function list(dir: Path): string[] {
  let result = []
  for (let entry of storage.list(dir)) {
    if (/^[A-Z2-7]{32}\.original\.uvf$/.test(entry)) continue // hidden
    let m = entry.match(/^([A-Z2-7]{32})\.shortened\.uvf$/)
    if (m) {
      let name = utf8Decode(storage.read(dir + '/' + m[1] + '.original.uvf'))
      if (!isLong(name) || shortName(name) != m[1]) throw invalid // see Implementation Notes
      result.push(name)
    } else {
      result.push(entry)
    }
  }
  return result
}

function create(path: Path, name: string) {
  if (isLong(name)) storage.writeFile(path + '/' + sidecarName(name), utf8(name)) // sidecar first
  storage.create(path + '/' + storedName(name))
}

function delete(path: Path, name: string) {
  storage.delete(path + '/' + storedName(name)) // node first
  if (isLong(name)) storage.delete(path + '/' + sidecarName(name))
}

function move(srcDir: Path, srcName: string, dstDir: Path, dstName: string) {
  if (isLong(dstName)) storage.writeFile(dstDir + '/' + sidecarName(dstName), utf8(dstName))
  storage.move(srcDir + '/' + storedName(srcName), dstDir + '/' + storedName(dstName)) // node moves atomically
  if (isLong(srcName)) storage.delete(srcDir + '/' + sidecarName(srcName))
}
```

Reading, writing, and other operations on an existing node only map the name via `storedName()`.

> [!TIP]
> Once verified, the contents of a `.original.uvf` file can be cached indefinitely, keyed by its hash. The hash determines its sidecar's content, so the mapping never changes. A cache entry only says what name a hash maps to. It doesn't say whether that node still exists.

The sidecar is always written before and deleted after the node. An interrupted operation therefore leaves at most an orphaned sidecar, which is harmless and MAY be deleted.

A move is never a plain pass-through. The target name is evaluated anew, and a move to another directory changes the ciphertext name anyway, because names are bound to their parent. This results in one or two files at the destination.

## Sync Conflicts

* **Sidecars**: concurrent creation of the same name produces identical sidecar contents, so conflicting copies are interchangeable and MAY be deleted.
* **Nodes**: a sync service may produce e.g. `{hash}.shortened (conflicted copy).uvf`. The middleware cannot resolve this alone, as resolving requires a new cleartext name. Instead, it reports such entries to the upper layer as the original name, with the same suffix inserted before the last extension. The upper layer then handles the entry like a conflict on an unshortened name:

```ts
let m = entry.match(/^([A-Z2-7]{32})\.shortened(.+)\.uvf$/)
// read and verify sidecar m[1] as in list(), then:
let conflictName = stem(name) + m[2] + extension(name) // e.g. "cAp6…0G (conflicted copy).uvf"
```

Renaming the conflicting node through the middleware converts it back into regular shortened form.

## Implementation Notes

* **Consistency.** If shortening is enabled, an application MUST shorten a name if and only if `isLong(name)`. If `nameShorteningThreshold` is absent or `-1`, an application MUST NOT shorten names, and `*.shortened.uvf` and `*.original.uvf` have no special meaning.
* **Sidecar verification.** When reading a sidecar, the middleware MUST verify that its content is valid UTF-8, that `isLong(content)` holds, and that `shortName(content)` equals the hash in the entry's name. Otherwise the entry MUST be treated as invalid.
* **Canonical form.** A stored name longer than `T` MUST be treated as invalid. Hence every name has exactly one stored representation.
* **Missing sidecar.** A `{hash}.shortened.uvf` without a matching sidecar cannot be mapped back to its name. It MUST NOT be listed as a regular node, but applications MAY offer recovery of its contents.
* **Reserved names.** Short names produced by the upper layer MUST NOT match `*.shortened*.uvf` or `*.original*.uvf`. This holds for all current name formats, as base64url contains no `.`.

## Security Considerations

* **Name swapping.** Sidecars are unauthenticated. Without sidecar verification, an adversary could swap the sidecars of two long-named siblings, and both names would decrypt successfully. Verification reduces this attack to finding a second preimage of the truncated SHA-256.
* **Leakage.** The sidecar reveals the ciphertext name, which would be visible on storage anyway without shortening. Shortening therefore leaks nothing beyond the name length, which is already observable.
* **Collisions.** 160 bits give a negligible accidental collision probability among the names of a directory (birthday bound 2^-80). Crafted collisions gain an adversary nothing: the colliding name would need to be a valid ciphertext, which the name format's authentication prevents.
* **Residual risk.** Losing a sidecar makes the node's name unrecoverable, because the hash is one-way.

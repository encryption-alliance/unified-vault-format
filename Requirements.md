# Design Requirements

## 1. Encrypt files individually
This format is intended to be used in scenarios where directory structures are synced (e.g. to remote storage). Since we can not make assumptions about the tools used for such job, we can not require them to deeply inspect file contents for changes. This means that files are not to be put into some kind of container format.

## 2. Atomic operations
To maximize security, we must not only care about confidentiality, but also integrity and availability. In order to guarantee file integrity, especially when not making assumptions about the timing of third party file access, we want I/O operations to remain atomic if the caller expects it. For example, renaming a folder should not require changing its children.

## 99. TBD
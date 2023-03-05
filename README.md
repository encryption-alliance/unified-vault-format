# New Encryption Format
:warning: Temporary repo until we found a name for a new vendor-independent FOSS encryption format

## Goals
* Define a common vendor-independent standard for encrypted directories on a per-file basis
* Discuss pros and cons of file formats and cipher choices
* Make the format be as simple as possible and the spec as clear as possible
  * Keep MAY or SHOULD requirements to a minimum to avoid diverging feature sets of different implementations
  * Easier implementation means fewer bugs and incompatibilities
  * Lean on the underlying file system as much as possible
* Create testcases to verify compliance to the format

## Non-Goals
* Invent any new ciphers or cryptographic operations
* Define container formats (such as encrypted block devices)
* Create an application

## Why?
* Give users freedom of choice regarding their preferred software
* Ease migration between different file encryption applications
* Simplify development and auditing of software

## Contribute
* :memo: Read about of [Design Requirements](Requirements.md)
* :speech_balloon: Join our [discussions](https://github.com/cryptomator/new-encryption-format/discussions)
* :speaking_head: Invite other tool vendors to join the effort
* :stethoscope:	Security audits and enhancements are highly welcome :wink:

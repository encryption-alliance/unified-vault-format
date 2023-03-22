# New Encryption Format
:warning: Temporary repo until we [found a name (please join this discussion)](https://github.com/cryptomator/new-encryption-format/discussions/3) for a new vendor-independent FOSS encryption format

## Goals
* Define a common vendor-independent standard for encrypted directories on a per-file basis
* Discuss pros and cons of file formats and cipher choices

## Non-Goals
* Invent any new ciphers or cryptographic operations
* Define container formats (such as encrypted block devices)
* Create an application

## Why?
* Give users freedom of choice regarding their preferred software
* Ease migration between different file encryption applications
* Simplify development and auditing of software

## Who is supporting this?
The standardization process is supported by some well-known FOSS products that share the common vision:
* [Cryptomator](https://github.com/cryptomator/cryptomator)
* [Cyberduck](https://github.com/cyberduck/)
* [gocryptfs](https://github.com/rfjakob/gocryptfs/)
* [ownCloud](https://github.com/owncloud)
* [rclone](https://github.com/rclone/rclone/)

## Contribute
* :memo: Read about of [Design Requirements](Requirements.md)
* :speech_balloon: Join our [discussions](https://github.com/cryptomator/new-encryption-format/discussions)
* :speaking_head: Invite other tool vendors to join the effort
* :stethoscope:	Security audits and enhancements are highly welcome :wink:

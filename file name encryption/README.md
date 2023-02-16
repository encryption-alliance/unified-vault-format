# File Name Encryption

:warning: this is a working draft

| Format ID | Description | Pros | Cons |
|---|---|---|---|
| NONE | Don't encrypt file names, just append a file extension | no issues | :warning: no privacy |
| AES-SIV-BASE64URL | Encrypt using AES-SIV, then base64url-encode | no fancy characters | 16 byte overhead<br>4/3 expansion |
| AES-SIV-BASE32-CI | Encrypt using AES-SIV, then base32-encode, apply case information on encoded ciphertext | no fancy characters<br>case-insensitive | :warning: leaks case information <br>16 byte overhead<br>8/5 expansion |
| AES-SIV-BASE4K | Encrypt using AES-SIV, then base4k-encode | short file names (in terms of chars) | 16 bytes overhead<br>unicode required |
| ... | ... | ... | ... |
# secure-vault-pwa

That project is about a PWA based password manager, inspired by [gauravchl/secure-wallet](https://github.com/gauravchl/secure-wallet).

**Currently, no dev was initiated, it’s only some reflexions about implementation, especially about cryptography design.**

Passwords and sensitive data are encrypted before stored in [browser localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API).
It’s named the « vault ».
However, since localStorage can easily be wiped out, encrypted data is also stored on a server after each change.
When app is loaded, last encrypted data is fetch from server and decrypted locally after user gave its master password.
Passwords and sensitive data are *never* stored in localStorage unencrypted.

That project is or will be composed of theses repositories :
- [secure-vault-pwa](https://github.com/paulgreg/secure-vault-pwa) : the Progressive Web Application,
- [secure-vault-api](https://github.com/paulgreg/secure-vault-api) : browser API about cryptgraphy & localStorage access,
- [secure-vault-server-node](https://github.com/paulgreg/secure-vault-server-node) : server code in node,
- secure-vault-web-extension : browser web extension.

## Planned features
- Progressive Web App : works on any modern browser, on mobile or desktop, offline support
- Sensitive data are encrypted via AES with a 256 bits key size (see below for cryptography details) ; argon2 is used as a key derivation function
- Ability to change your master password
- Strong and random password generation via [generate-password](https://github.com/brendanashworth/generate-password)
- Ability to save notes, not only passwords
- Keep trace of each password change date
- Ability to search by title or changed date
- Vault synchronized on server (allowing use on multiple device, allowing vault restoration even if browser’s data has been cleared or you change your device)

### Known issues
- conflict are not handled (at least for now) so do not change data on 2 devices simultaneously : save first on device A then reload app on device B

### Next 
- Browser extension to allow quicker way to log into a website
- Conflicts handling / warning if data on server is more recent

## Cryptography design
- A salt `S` is generated by [Crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues),
- User master password is used in argon2id, with a per-user salt named `S` to generate a key named `K`,
- An initialization vector `IV` is generated by [Crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues),
- Data (a.k.a. vault containing passwords) are encrypted via AES-256-GCM with key `K` and `IV`,
- GCM produces a message authentication code `AC`, allowing us to check if password is correct or data hasn’t been tampered with.

Master password and key `K` are *never* stored in localStorage, they only exists in memory.

`S`, `IV` and `AC` are stored in localStorage (in clear text).
Per-user random salt is here to prevent pre-computed attacks, a.k.a rainbow tables.

### Use case : « log in » and access encrypted data
1. Last vault version (and `S`, `IV` and `AC`) is fetched from server, replacing data in localStorage if most recent,
2. User types its master password,
3. We use argon2id with the master password and `S` to generate key `K`,
4. We decrypt the vault with AES by using key `K` and `IV` and check message integrity with `AC`.

### Use case : creation
1. User master password entropy is tested : it should be strong enough to continue process,
2. A salt `S` is generated by [Crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues),
3. An initialization vector `IV` is generated by [Crypto.getRandomValues](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues),
4. We generate key `K` via argon2id with the master password and the salt `S`,
5. We create an empty vault object, encrypt it with `K`, resulting in a encrypted vault and an authentication code `AC`,
6. We store `S`, `IV` and `AC` in localStorage,
7. We register to server by sending username, encrypted vault, `S`, `IV` and `AC` 
8. Server send us a secret, displayed to user (and ask her/him to write it down and store it safely) ; secret is stored in localStorage.

## Data in localStorage
- encrypted vault
- AES parameter `IV` and output `AC` (in clear text)
- argon2 parameters : salt `S` (in clear text), number of iterations, parallelism, used memory, hash length
- version number, incremented after each change
- username, used to store data on server
- secret key generated from server the first time (it is needed to upload data)

## Data on server
After each change, vault is encrypted, stored in localStorage and uploaded to server.
Vault is never overwritten but wrote « by version » to avoid any data loss.

Note that conflict resolution isn’t handled for now (but since data is never overwritten, we could « manually » restore a previous version).

When app is loaded, last version of vault is fetched from the server and will replace localStorage if more recent.

A secret is randomly generated by the server on the first upload, needed for future upload (to avoid an attacker overwrites user’s data)

If you want to load data from another device or if the localStorage was emptied, you will need to provide the username and the secret key.

Data are stored in simple directory / json files on server :
```
data/
   username/
       information.json # secret key, last version number
       version1.json    # encrypted data, AES & argon2 parameters
       version2.json    # encrypted data, AES & argon2 parameters (so they could be changed from one to another version)
       ...
 ```

## Vault composition
Vault contains a list of item.
Each item contains these fields :
- title, mandatory : usually an URL but may be any alphanum characters (for « offline accounts »)
- password, optional,
- username, optional,
- notes, optional,
- password_last _change : automatically updated each time password is changed

## Technological Stack
- [Web Cryptography API](https://www.w3.org/TR/WebCryptoAPI/) (random, AES)
- [antelle/argon2-browser](https://github.com/antelle/argon2-browser/) (argon2id) ?
- [brendanashworth/generate-password](https://github.com/brendanashworth/generate-password) (password generation)
- [donburks/password-entropy](https://github.com/donburks/password-entropy) (password entropy check) ?
- react
- material-design ?
- server in node

## Risks - points of attention
- bad crypto design
- keys, master password or sensitive data stored in clear text or send via network
- bad random choice for IV, salt or passwords : should be prevented by use of native browser crypto API
- bad usage of AES  (like bad parameters choice)
- bad usage of Argon2 (like bad parameters choice)
- bad AES implementation : should be prevented by use of native browser crypto API
- security problem in Argon2 implementation
- external script on page stealing sensitive data : no external resources should be loaded ; use of [subresource integrity](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) and (content security policy)[https://developer.mozilla.org/fr/docs/Web/HTTP/CSP)[https://developer.mozilla.org/fr/docs/Web/HTTP/CSP]
- sensitive data leaked in browser
- difficulty to make sure sensitive data are free in memory
- server : user data to be overwritten
- server : bad string input validation
- server : directory traversal

## Resources
- https://cryptobook.nakov.com/
- https://crypto.stackexchange.com/
- https://diafygi.github.io/webcrypto-examples/
- https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto
- https://github.com/mdn/dom-examples/tree/master/web-crypto
- https://1password.com/files/1Password-White-Paper.pdf
- https://www.ise.io/casestudies/password-manager-hacking/

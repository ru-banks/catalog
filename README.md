# UniBank bank catalog

Signed catalog and encrypted bank specs for the [UniBank](https://github.com/and7ey/unibank)
client (ADR-0009). This repository is the public distribution point; plaintext
sources and private keys live outside it.

## Layout

```
catalog.json       exact bytes that are signed
catalog.json.sig   detached Ed25519 signature (base64url)
specs/<id>-<ver>.ubx  XChaCha20-Poly1305 artifact (AAD: "<id>@<version>")
logos/<id>.<ext>   optional, hash-pinned in the catalog
```

## How the app uses it

1. Fetch `catalog.json` and verify the signature with the public key embedded in
   the app.
2. Pick a bank, download its `.ubx`, check `sha256`/size from the catalog.
3. Decrypt with the key for `keyId` and validate against the spec schema.

The catalog carries no user data. Rebuilds are produced by
`tools/spec/publish.ts` in the client repository; artifacts are immutable, a
rollback is a new catalog committed on top.

## Notice

Bank names and protocols belong to their owners. Specs are published encrypted
to keep them out of search engines and forks; the encryption key ships with the
client, so this is obfuscation, not secrecy. Integrity is guaranteed by the
signature.

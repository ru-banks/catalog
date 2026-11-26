# Unibank bank catalog

[Русский](README.md)

Signed catalog and encrypted bank specifications for the Unibank client. The app
downloads the catalog and the specifications from this repository. Plaintext
sources of published banks and private keys live in another repository.

## Layout

```
index.json            the file that is signed
index.json.sig        signature for it, base64url
specs/<id>-<ver>.ubx  encrypted bank specification
logos/<id>.<ext>      bank logo, if available
docs/                 documentation for people who write specifications
```

## Adding a bank

A bank specification describes how to sign in to the bank, which requests to
send, how to map the response and which responses are errors. It is a JSON file.
There is no bank code in the app; the engine runs the specification.

Anyone can write one:

1. Open an issue called "New bank" and describe what already works.
2. Write the specification using [docs/README.md](docs/README.md). An AI agent
   makes this faster: it downloads the APK, traces the protocol and drafts the
   specification. The workflow and a ready prompt are in
   [docs/ai-agent.md](docs/ai-agent.md).
3. Send a pull request with the plaintext specification and protocol notes.
4. After review we publish the bank in the catalog.

Documentation (in Russian):

- [docs/ai-agent.md](docs/ai-agent.md): building a specification with an AI
  agent, plus the ready prompt;
- [docs/process.md](docs/process.md): the path from issue to publication;
- [docs/spec-v1.md](docs/spec-v1.md): the specification language;
- [docs/capabilities.md](docs/capabilities.md): what is required and what is
  optional;
- [docs/checklist.md](docs/checklist.md): evidence to attach to a pull request;
- [docs/examples/bank1.spec.json](docs/examples/bank1.spec.json): a demo bank
  specification.

Issues and pull requests are public. Do not put tokens, cookies, account
numbers, card data or personal data there.

## How the app reads the catalog

1. Downloads `index.json` and verifies the signature with the key embedded in
   the app.
2. Downloads the bank's `.ubx` and checks its hash and size against the catalog.
3. Decrypts the specification and validates it against the schema.

The catalog contains no user data. It is built by `tools/spec/publish.ts` in the
client repository. Artifacts never change: a rollback is a new catalog on top.

## Notice

Bank names and protocols belong to their owners. Specifications are stored
encrypted so they do not show up in search engines and forks. The decryption key
ships with the client, so this is obfuscation, not secrecy. The signature
guarantees integrity.

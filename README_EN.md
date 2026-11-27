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

## Supported banks

The table matches the catalog as of 27 November 2026; exact versions and
capability lists are in [`index.json`](index.json). All banks are **beta**: the
app reads your own accounts, cards and transactions, but does not move money.

`✓` — available in the published specification, `—` — not available.

| Bank | Balances | Cards | Transaction history | Cashback and category selection | Cashback history | Limitations |
|---|:---:|:---:|:---:|:---:|:---:|---|
| **T-Bank** | ✓ | ✓ | ✓ | ✓ | — | Live sign-in has not been tested yet — only previously recorded responses and a test flow. SBP transfers are disabled. |
| **VTB** | ✓ | ✓ | ✓ | ✓ | — | SMS sign-in. The bank may not accept automatic session refresh, then a passcode or a new sign-in is required. Cashback categories depend on the month and product. |
| **Alfa-Bank** | ✓ | ✓ | ✓ | ✓ | ✓ | Sign-in needs a human: phone, secret code and SMS; anti-fraud may ask to verify identity. Selecting "Orange" categories has not been tested with a live write. |
| **MTS Dengi** | ✓ | ✓ | ✓ | ✓ | — | Only the MTS Dengi bank is supported. MTS Bank is a separate bank; its accounts and cards, including the "MTS Dengi" card, are not read. Sign-in goes through the MTS ID web cabinet; the session lives about an hour and automatic refresh is not enabled yet, so a new sign-in is required after it expires. |
| **OTP Bank** | ✓ | ✓ | ✓ | — | — | The first sign-in from a new device requires identity verification (passport, card or agreement). The bank's protection against frequent requests blocks bursts — a calm pace is needed. |
| **Freedom Bank** | ✓ | ✓ | — | — | — | Transaction history is closed until verification is completed: without it the bank returns `401`. There is no automatic session refresh — sign in again by SMS. |
| **Finuslugi** | ✓ | — | ✓ | — | — | The bank does not issue cards. History is available only for the wallet; deposits are shown without history. Credit products have not been researched. |
| **DOM.RF Bank** | ✓ | ✓ | ✓ | ✓ | — | First sign-in is password + OTP; automatic refresh is bound to the device, so a new device needs a new sign-in. The Mincifry certificate must be installed on the device. Cashback for past months is not available via a separate API. |
| **Ozon Bank** | ✓ | ✓ | ✓ | ✓ | — | Sign-in opens on the bank's page: Ozon ID and the bank passcode. Transaction history is returned in chunks of up to 50 records. Cashback for past months is not read in the published version. |
| **SberBank** | ✓ | ✓ | ✓ | ✓ | — | Sign-in by card or SMS on the bank's page. The session is short and is refreshed with a device-bound digital passcode. After several errors the bank enables a captcha. |
| **Yandex Pay** | ✓ | ✓ | — | — | — | Sign-in through the Yandex ID page with push 2FA. Transaction history is not supported in the published version — only balances and cards. |
| **Cifra Bank** | ✓ | ✓ | ✓ | — | — | Live sign-in was tested on an account without cards or operations: card and history field mapping is confirmed by the bank's code, not by live data. Sign-in by card number has not been tested. |

SBP transfers are not included in the published specifications yet: the app only
reads data. That is the next step, not a hidden feature.

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

## Why the specifications are encrypted

A specification is a ready-made recipe: which requests to send, which headers and
signatures are needed, how to parse the response. In plaintext it is useful not
only to developers, so the catalog stores an encrypted `.ubx` instead of the
source JSON.

- **App stores.** Some supported banks are under sanctions. A public repository
  with ready recipes for their internal APIs is extra material for Apple and
  Google review. Encryption does not hide the fact that a bank is supported, but
  it keeps the ready recipe out of search and forks.
- **The banks themselves.** Banks change internal APIs and tighten anti-fraud.
  Plaintext specifications are easier to find, compare with traffic and close.
  An encrypted artifact raises the cost of that work and gives us time to ship an
  update through the catalog instead of an app release.
- **Search and forks.** `.ubx` files are not indexed by search engines and do not
  spread through forks as source.

This is not a guarantee of approval in the App Store or Google Play: store policy
towards apps that use banks' internal APIs remains a grey area. Encryption only
reduces the risk that a ready recipe becomes a public argument against the app.

To be clear: **this is obfuscation, not secrecy.** The decryption key ships with
the client, so a determined person can still extract the specification. Integrity
is guaranteed not by encryption but by the `index.json` signature and artifact
hashes: the app will not accept a substituted catalog.

## Notice

Bank names and protocols belong to their owners. The catalog does not claim
official support from any bank and is not affiliated with them. Use it only with
your own accounts and at your own risk.

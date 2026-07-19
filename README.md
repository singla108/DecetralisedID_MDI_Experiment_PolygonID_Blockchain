# Polygon ID Verifiable Credential POC

A proof of concept for issuing and verifying a custom **self-sovereign identity credential** — "MDI student" — using [Polygon ID](https://polygonid.com/) (now [Privado ID](https://www.privado.id/)), decentralized identifiers (DIDs), zero-knowledge proofs, and the Polygon Mumbai testnet.

The POC issues a verifiable credential asserting `student: true` to a holder's Polygon ID DID, stores it in the Polygon ID mobile wallet, and verifies it via a zero-knowledge proof — without ever revealing the holder's private key or any data beyond what the verifier explicitly requested.

## What's in this repo

```
.
├── README.md                      # This file
├── credentials/
│   ├── MDI-student.schema.json    # JSON Schema for the "MDI student" verifiable credential
│   └── MDI-student.jsonld         # JSON-LD context defining the custom `student` claim
└── docs/
    ├── setup-issuer-node.md       # How to stand up the Polygon ID issuer node used in this POC
    ├── ngrok-notes.md             # Exposing the local issuer node to the mobile wallet via ngrok
    └── auth-flow.md               # What the wallet/verifier actually do under the hood (ZK auth flow)
```

## How it works

1. **Issuer node.** A self-hosted instance of the official [Polygon ID issuer node](https://github.com/0xPolygonID/issuer-node) runs locally via Docker Compose (Postgres, Redis, HashiCorp Vault for key storage), with its own DID generated on Polygon Mumbai. See [`docs/setup-issuer-node.md`](docs/setup-issuer-node.md).
2. **Credential schema.** A custom credential type, `MDI student`, is defined via a JSON Schema + JSON-LD context pair ([`credentials/`](credentials)). It has a single custom claim: `student` (boolean) — "you are from MDI".
3. **Issuance.** Using the issuer node's UI/API, a credential is issued to a specific holder DID, asserting `student: true`.
4. **Holding.** The holder scans a QR code with the [Polygon ID mobile wallet](https://polygonid.com/wallet), which pulls the credential from the issuer node and stores it locally — the credential and the holder's private key never leave the device.
5. **Tunneling.** Since the issuer node runs on a local machine, [ngrok](https://ngrok.com/) exposes it over HTTPS so the mobile wallet (on a separate device/network) can reach it. See [`docs/ngrok-notes.md`](docs/ngrok-notes.md).
6. **Verification.** A verifier requests proof of the `student` claim. The wallet generates a zero-knowledge proof (Groth16, `authV2` circuit) proving it holds a valid, non-revoked credential satisfying the request — without disclosing the credential itself. See [`docs/auth-flow.md`](docs/auth-flow.md) for the full technical sequence, reconstructed from captured debug logs.

## Key concepts demonstrated

- **Decentralized identifiers (DIDs):** `did:polygonid:polygon:mumbai:...`, with privacy-preserving per-relationship profile DIDs derived from a single root identity.
- **Verifiable credentials:** custom-schema credentials issued and anchored via a Polygon ID issuer node.
- **Zero-knowledge proofs:** the `authV2` circuit (Groth16) proves control of a DID and possession of valid claims without revealing underlying secrets.
- **On-chain state anchoring:** identity state (claims tree, revocation tree) is committed to a Global Identity State Tree (GIST) smart contract on Polygon, so verifiers can check an identity/credential hasn't been revoked without trusting a central server.
- **iden3comm protocol:** the messaging protocol (`https://iden3-communication.io/`) used between wallet, issuer, and verifier for auth and credential exchange.

## Prerequisites

- Docker and Docker Compose
- [ngrok](https://ngrok.com/) account (free tier is sufficient)
- The [Polygon ID mobile wallet app](https://polygonid.com/wallet) (iOS/Android) — now rebranded as the Privado ID wallet
- An Ethereum-compatible private key funded with Polygon Mumbai (or current Amoy) test MATIC, for the issuer's identity

## Getting started

Follow [`docs/setup-issuer-node.md`](docs/setup-issuer-node.md) to stand up the issuer node, then [`docs/ngrok-notes.md`](docs/ngrok-notes.md) to expose it to the mobile wallet. Once running, use the issuer UI to import the schema in [`credentials/`](credentials) and issue an `MDI student` credential to a test DID.

## Status & known limitations

This is a hackathon/exploration-stage POC, not production code:

- Built against **Polygon Mumbai**, which was deprecated in 2024 in favor of **Polygon Amoy**. Network config, RPC URLs, and the identity state contract address need updating to reproduce this today — see the note in [`docs/auth-flow.md`](docs/auth-flow.md).
- Relies on ngrok for public reachability, which is fine for a demo but not a deployment story.
- No verifier application code is included here — this repo documents the issuer-side setup and the observed wallet/verifier protocol flow, based on the original POC's working notes and captured debug logs.
- Polygon ID has since been rebranded to **Privado ID**; check [docs.privado.id](https://docs.privado.id/) for current tooling, as some URLs/tools referenced here (e.g. `push-staging.polygonid.com`) may have moved.

## Background reading

The original working files from this POC (detailed write-up, config walkthroughs, enterprise overview deck, verifier reference PDF, and screenshots) are kept alongside this repo's source material for reference and can be added to `docs/` as needed.

## License

MIT — see [LICENSE](LICENSE).

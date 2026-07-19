# Authentication / verification flow observed in this POC

Reconstructed from the captured debug logs (`Polygon ID Stack trace.txt`, `StackTrace.txt`). This documents what the Polygon ID mobile wallet actually does when authenticating against a verifier, for reference when debugging the flow.

## Network

- **Blockchain:** Polygon
- **Network:** Mumbai testnet (deprecated in favor of Amoy since this POC was built — see [Notes](#note-on-mumbai-deprecation))
- **RPC:** `https://polygon-mumbai.infura.io/v3/...`
- **Identity state contract:** `0x134B1BE34911E39A8397ec6289782989729807a4`
- **Push notification service:** `https://push-staging.polygonid.com/api/v1`

## DID format

Identities are `did:polygonid:polygon:mumbai:<base58-identifier>`, e.g.:

```
did:polygonid:polygon:mumbai:2qFsLvZrMaB5Lxd3m3uPgHxz3vNWR97czko4UqmBcx
```

The wallet also derives per-relationship **profile DIDs** from the root identity (a privacy feature so the same holder can present a different DID per verifier/issuer relationship).

## High-level sequence

1. **`AuthenticateUseCase`** kicks off authentication with a verifier/issuer via the [iden3comm](https://iden3-communication.io/) protocol.
2. The wallet builds a DID Document (`did_doc`) for itself, including a `push-notification` service endpoint (RSA-OAEP-512 encrypted device token) so the verifier can push messages back.
3. **`GetAuthChallengeUseCase`** generates a numeric challenge that the wallet must prove knowledge of a private key against.
4. **`LoadCircuitUseCase`** loads the `authV2` zero-knowledge circuit (Groth16 proving system — see the JWZ header `"alg":"groth16","circuitId":"authV2"`).
5. **`GetIdentityAuthClaimUseCase`** / **`GetAuthClaimUseCase`** fetch the identity's auth claim (an `iden3` claim encoding the holder's public key) from the identity's local claims tree.
6. **`GetGenesisStateUseCase`** / **`GetLatestStateUseCase`** compute the identity's genesis state and latest on-chain state (Merkle roots: `claimsRoot`, `revocationRoot`, `rootOfRoots`, aggregated into `state`).
7. **`GetGistMTProofUseCase`** fetches a Merkle proof against the **Global Identity State Tree (GIST)** — proof that this identity's state is included in the on-chain global state tree (`idStateContract`).
8. **`GetAuthInputsUseCase`** assembles all of the above (auth claim, incremental/non-revocation proofs, tree state, GIST proof, challenge, signature) into the private/public inputs for the `authV2` circuit.
9. **`calculateWitness`** runs the circuit locally on-device to produce a zero-knowledge proof — the wallet never reveals its private key, only a proof that it knows one corresponding to the claimed DID and that the DID's state is anchored on-chain.
10. The resulting proof is packaged as a JWZ (JSON Web Zero-knowledge token) and sent back to the verifier as the `authResponse`, completing authentication without exposing the underlying claim data.

## Why this matters for the POC

The same `authV2`-based ZK proof mechanism underlies both:
- **Authentication** (proving control of a DID), and
- **Credential presentation** (proving possession of a valid, non-revoked `MDI student` credential matching a verifier's query) — the same Merkle-proof/circuit machinery, just with the credential's claim substituted for the bare auth claim.

## Note on Mumbai deprecation

This POC was built while Polygon's Mumbai testnet was still active (2023). Mumbai was deprecated in 2024 in favor of **Polygon Amoy**. Reproducing this POC today requires updating the network config (`network: mumbai` → `amoy`), RPC URLs, and the identity state contract address to their Amoy equivalents — check the current [Privado ID / Polygon ID docs](https://docs.privado.id/) for up-to-date values.

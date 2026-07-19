# ngrok notes

The Polygon ID issuer node needs a publicly reachable HTTPS URL so the Polygon ID mobile wallet can scan a QR code and call back to it (to fetch a credential offer, or to submit an auth/proof response). During local development this POC used [ngrok](https://ngrok.com/) to tunnel the locally running issuer node.

> The original detailed write-up of the ngrok configuration changes lives in `Ngrok changes.docx` (kept alongside this repo's source docs) — this file summarizes the key points needed to reproduce the setup.

## Why it's needed

- The issuer node UI generates QR codes containing a callback URL (e.g. an `iden3comm` request).
- If that URL points at `localhost`, the mobile wallet (running on a physical device on a different network) can't reach it.
- ngrok maps a public `https://<subdomain>.ngrok-free.app` (or similar) URL to the locally running issuer node/API port.

## General flow

1. Start the issuer node locally (see `setup-issuer-node.md`).
2. Start an ngrok tunnel pointing at the issuer node's exposed port (e.g. port `80` for the UI/API gateway):
   ```bash
   ngrok http 80
   ```
3. Take the `https://...ngrok-free.app` forwarding URL ngrok prints out.
4. Update the issuer node's environment configuration (`.env-issuer`, `.env-api`, or equivalent `ISSUER_SERVER_URL` / public URL settings) to use that ngrok URL instead of `localhost`.
5. Restart the affected containers so the new public URL is picked up in generated QR codes / callback links.
6. Re-scan the QR code from the Polygon ID wallet app — the wallet's requests will now route through the ngrok tunnel to the local issuer node.

## Gotchas encountered

- ngrok's free tier URL changes on every restart, so the issuer node's public URL config needs to be updated and the relevant containers restarted each time the tunnel is recreated.
- Requests through the tunnel must be HTTPS; make sure the mobile wallet is hitting the `https://` forwarding URL, not `http://`.
- If the issuer node caches its public base URL at startup, a config change alone isn't enough — the container needs a restart, not just an env file edit.

# Amazon Flex — Request Attestation & Signing Toolkit

> A complete, working reverse-engineering of the Amazon Flex Android app's device-attestation and HTTP request-signing scheme (the current **V2** protocol) — able to register a device and produce server-accepted signed API requests outside of the app.

---

## Overview

Modern Amazon Flex ("Rabbit") protects its backend APIs with a hardware-backed **device attestation** layer and a per-request **HTTP Message Signature**. A plain HTTP client cannot talk to these endpoints: every request must carry a cryptographic signature produced by a key that Amazon registered after verifying the device, and that signature must be built byte-for-byte the way the app builds it.

This project documents and reproduces that entire scheme. It takes the attestation and signing logic buried inside the app and turns it into a clean, repeatable workflow:

1. **Register** a signing key with Amazon's attestation service, and
2. **Sign** arbitrary API requests with that key so they are accepted by production endpoints.

It is the result of full static analysis of the app plus a validated, runnable implementation — not a theoretical write-up. The signing pipeline mirrors the app's own request signer and attestation service exactly, down to component ordering and challenge generation.

**Who this is for:** teams building automation, monitoring, or integration tooling against Flex-style endpoints who need to understand — and reproduce — the attestation and request-signing handshake that otherwise blocks all off-device access.

---

## The problem it solves

| Without this toolkit | With this toolkit |
|---|---|
| Flex endpoints reject any request not signed by a registered, hardware-attested key. | A registered key is obtained and persisted automatically. |
| The V2 signing scheme (component list, challenge, digest, parameter order) is undocumented and easy to get subtly wrong — one wrong byte fails the signature. | Signing is reproduced exactly from the app's own logic and verified against a live content-digest check. |
---

## Key features

- **End-to-end V2 attestation flow:** challenge retrieval, hardware key generation, request-hash computation, Play Integrity token acquisition, and registration, exactly as the app performs them.
- **Faithful V2 request signing:** reproduces the app's signer: derived components (`@path`, `@method`, `@target-uri`), conditional header inclusion, content-digest, challenge generation, and the strict signature-parameter ordering.
- **Two independent implementations:**
  - **On-device:** generates and uses a genuine hardware-backed (StrongBox→TEE fallback) key inside a running app process, with a real Google Play Integrity token.
  - **Pure Python:** reconstructs the registration and signing math with standard cryptography primitives, including the Android Key Attestation certificate extension, for offline study and experimentation.

- **Built-in correctness guard:** the sender re-hashes the exact request body and compares against the signed content-digest, catching any mismatch before the server would.
- **Signature-only fast path:** once a key is registered, re-sign new requests in seconds without re-attesting (which would otherwise invalidate the key).

---

## How it works

**Registration:**
`GET /v2/challenge` → generate a hardware-attested EC P-256 key bound to the challenge → compute a SHA-256 request hash over the certificate chain → obtain a Play Integrity Standard token for that hash → `POST /v2/android/attestations/register` → receive and persist the `keyId` / `region`.

**Signing:**
load the registered hardware key → build the signature base (ordered components + signature parameters) → sign with `SHA256withECDSA` → emit the `Signature-V2`, `Signature-Input-V2`, and content-digest headers.

**Sending:**
assemble the production request with the freshly signed values and the exact signed body, verify the content-digest matches the body, and POST to the live endpoint.

Day-to-day, only signing and sending are needed; the registered key is long-lived, and re-signing is fast.

---

## Main use cases

- **Automated API access:**  register once, then programmatically produce signed, server-accepted requests.
- **Integration & monitoring tooling:** build clients that interact with attested endpoints from a backend rather than a phone.

---

## Security, limitations & important considerations

- **Bring your own credentials.** The toolkit does not include or distribute Amazon access tokens, signing keys, or private endpoints. It operates only with values you supply.
- **Signatures are single-use.** Each carries a fresh challenge and timestamp; sign immediately before sending, and do not replay.
- **On-device requirements are real.** Hardware-backed key generation and a genuine Play Integrity token require an appropriately provisioned Android device/emulator; the pure-Python path is for research and will only produce server-accepted signatures with a legitimately registered key.
- **Protocol-version specific.** It targets the app's current V2 scheme; Amazon may change the protocol, after which the implementation would need to be re-aligned (a service the author can provide).

---

## Contact

**Interested in this project, need a customized version, or want to discuss purchasing the source code?**

Contact me on Telegram: [t.me/roulia_umar](https://t.me/roulia_umar)

I can also help adapt the toolkit to other endpoints, keep it aligned with protocol changes, or integrate it into your own systems.

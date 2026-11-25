# Project 1 — Unified Crypto Wallet
## Security review: API keys, wallet secrets, and network data flow

**Review date:** 2026-11-25  
**Repository reviewed:** `GitHub2543/MEWconnect-web-client` (default branch `master`)  
**Project context:** Project 1 is the Unified Crypto Wallet: a read-only multi-chain dashboard for BTC/ETH/SOL. The agreed integration direction is to preserve the existing MEWconnect core and add multi-chain read-only functionality as a separate module.  
**Review type:** Static source-code review of the repository snapshot; not a live-device, production, or network-traffic assessment.

## Executive summary

- No verified API key, mnemonic, seed phrase, or private key was found being transmitted to the Monarch domain `monarch-member-bureau.web.app` in the source files reviewed.
- No direct code reference to that domain was found in the reviewed project context. This does **not** prove what happened on the user's Redmi device or what a third-party website collected.
- A concrete hardening concern exists: debug statements in the Ethereum provider log transaction objects, signed raw transactions, and an `eth_sign` result. Whether those logs are emitted depends on the runtime debug configuration. Disable production debug output and redact sensitive payloads.
- `WalletInterface` holds a private key in an in-memory object and exposes getter methods. This is not, by itself, evidence of network exfiltration or persistent storage, but it increases the importance of keeping the object out of logs, analytics, crash reports, and untrusted application contexts.
- This review did not establish that an API key was compromised. No credential values are included in this report.

## Scope and evidence

Reviewed files in `GitHub2543/MEWconnect-web-client`:

| File | Relevant observation |
|---|---|
| `src/connectProvider/web3Provider/WalletInterface.js` | Stores `_privKey` as `this.privateKey`; exposes `getPrivateKey()` and `getPrivateKeyString()`; signs messages locally with `ecsign`. |
| `src/connectProvider/web3Provider/web3-provider/methods/eth_sendTransaction.js` | Debug statements include the transaction object, transformed transaction, and `_response.rawTransaction`. |
| `src/connectProvider/web3Provider/web3-provider/methods/eth_sign.js` | Debug statement includes the signing result. |
| `src/connectProvider/web3Provider/web3-provider/methods/personal_sign.js` | Debug logging records request rejection; no secret value was observed in the matched lines. |
| `src/connectProvider/web3Provider/MEWconnect/index.js` | Uses `https://connect.mewapi.io` as the V1 signaling URL and relays signing requests through the MEWconnect session. |
| `src/connectProvider/web3Provider/web3-provider/providers/http-provider.js` | Routes JSON-RPC requests through middleware and an HTTP request manager. |

Source links:
- [WalletInterface.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/WalletInterface.js)
- [eth_sendTransaction.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/web3-provider/methods/eth_sendTransaction.js)
- [eth_sign.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/web3-provider/methods/eth_sign.js)
- [personal_sign.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/web3-provider/methods/personal_sign.js)
- [MEWconnect/index.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/MEWconnect/index.js)
- [http-provider.js](https://github.com/GitHub2543/MEWconnect-web-client/blob/master/src/connectProvider/web3Provider/web3-provider/providers/http-provider.js)

## Findings and severity

### F-01 — Sensitive transaction data can enter debug logs
**Severity:** Medium (conditional on debug output being enabled)

The transaction middleware contains debug calls for the raw transaction object, a transformed transaction, and the signed raw transaction before broadcast. The `eth_sign` handler also logs the returned signature result.

**Risk:** If debug namespaces are enabled in a production build or logs are collected by a host application, diagnostics could retain transaction details or signatures. This is not evidence that a third party received them.

**Recommended fix:**
1. Disable debug namespaces in production builds.
2. Remove logs of transaction objects, raw signed transactions, and signature results.
3. If diagnostics are essential, log only request IDs, method names, and sanitized error codes.
4. Add tests that fail if secret-bearing values are passed to the logger.

### F-02 — Private key is represented in a JavaScript object
**Severity:** Medium (design hardening; no exfiltration evidence)

`WalletInterface` stores a private key in `this.privateKey` and exposes getter methods. The reviewed code signs locally. The matched code does not establish that this key is sent to a remote endpoint.

**Recommended fix:**
1. Keep private-key handling in the smallest possible isolated component.
2. Avoid exposing raw private-key getters to general provider code; expose narrowly scoped signing operations instead.
3. Ensure key material is excluded from logs, analytics, telemetry, crash reports, and error serialization.
4. Add tests for accidental serialization and logging.

### F-03 — Signaling and JSON-RPC network paths need endpoint allowlisting
**Severity:** Medium (verification incomplete)

The MEWconnect client uses `https://connect.mewapi.io` for V1 signaling, while the HTTP provider delegates JSON-RPC requests to its request manager. This review did not complete an exhaustive trace of all endpoint configuration, redirect behavior, and payload handling.

**Recommended fix:**
1. Document the allowed signaling and RPC hosts.
2. Validate HTTPS and apply an explicit host allowlist where feasible.
3. Do not send mnemonics or raw private keys through signaling or JSON-RPC.
4. Add tests proving that only the intended public address, transaction request, and signed transaction data cross the relevant boundaries.

### F-04 — Monarch domain relationship not established
**Severity:** Informational / unconfirmed

No direct reference to `monarch-member-bureau.web.app` was identified in the project files reviewed. The user previously reported entering an email address and a public BTC address on that site, but did not connect a wallet, sign a message, or send BTC.

**Assessment:** The reported activity may expose the entered email and public address to the site operator. A public BTC address alone cannot authorize spending. The available evidence does not establish mnemonic/private-key theft, wallet control, or a link between the site and this repository.

## Companion repository context (separate codebase)

A previous static review of `ARMNOI/ecash-wallet-mobile` observed:
- Its documented design stores a wallet mnemonic through a platform-backed KeyStore (iOS Keychain / Android Keystore-backed storage) and derives extended private material at runtime.
- A BitWindow coin-news client uses a bearer token for a documented local backend endpoint. The endpoint-routing path should be verified to ensure that the token cannot be sent to an arbitrary remote host.
- A remote endpoint configuration service can influence public Electrum/Esplora-style service endpoints. This creates endpoint-integrity risk if the configuration source is compromised, but is not evidence of secret exfiltration.

These are observations from a separate repository and must not be treated as proof that the MEWconnect web client shares the same implementation or runtime behavior.

## API key status

**Status: Not confirmed compromised.** The source review described above did not establish a real exposed API key or a successful unauthorized use. This report deliberately contains no credential values. A source-code scan cannot determine whether an API key in a user's email, browser storage, device, or external service was used.

If a real credential is later identified:
1. Do not paste it into issues, chat, logs, or this repository.
2. Revoke or rotate it through the issuing provider.
3. Review provider audit logs and usage history.
4. Replace it with a least-privilege credential stored in an approved secret manager or CI secret store.
5. Remove it from the current source and address its historical exposure separately; deleting a visible string alone does not invalidate a leaked key.

## Action plan

- [ ] Remove or redact sensitive debug logging in `eth_sendTransaction.js` and `eth_sign.js`.
- [ ] Review all logger, analytics, crash-reporting, and error-serialization paths for wallet secrets.
- [ ] Reduce raw private-key getter exposure in `WalletInterface.js`.
- [ ] Trace the full signaling and HTTP request path, including host selection and redirects.
- [ ] Add tests ensuring mnemonics/private keys never enter network payloads or logs.
- [ ] Keep the multi-chain read-only module separate from MEWconnect signing/session code.
- [ ] For the read-only BTC/ETH/SOL dashboard, accept public addresses only; do not request seed phrases or private keys.
- [ ] Verify API keys through provider-side audit logs; never test or disclose credential values in source control.

## Limitations

This is a bounded static review of selected files and a previously reviewed companion repository. It does not inspect the user's live Redmi device, browser storage, Gmail account state, private repositories not available to the connected GitHub account, deleted Git history, production build artifacts, server-side logs, or actual network packets. Findings are not attribution of wrongdoing to any person or organization.

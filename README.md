# Phala on AntSeed — ownership and TEE evidence

Peer ID `88c849029c02310ce360625a9d8e333a50f515d6` (display name "Phala") is
operated by Phala. The public ownership proof in
[antseed.json](antseed.json)
and this DNS TXT record bind that identity to Phala's GitHub organization and
domain:

```text
_antseed.phala.com TXT "antseed-peer=88c849029c02310ce360625a9d8e333a50f515d6"
```

AntSeed seller node and the Phala gateway can both present TDX attestation;
the current integration does not verify each response end to end.

## What the seller sells

Phala confidential inference through the Phala-operated gateway, branded as
`inference.phala.com`. The seller currently reaches it through `tee.redpill.ai`.
Only compatible models marked `is_tee=true` in the gateway catalog are listed,
with category `confidential`. Advertised prices matched the gateway's public
catalog when checked on 2026-10-25; availability and prices can change.
These catalog declarations do not independently prove where inference runs.

## What a buyer can verify

Claim names below omit the `antseed-verifier:` prefix used in verification
results. These semantics refer to verifier source commit
[`8a94013c8746e114a2527483415b0eaf1ad23df2`](https://github.com/AntSeed/antseed-verifier/tree/8a94013c8746e114a2527483415b0eaf1ad23df2),
built into the deployed seller image as `@antseed/antseed-verifier` version
`0.1.0`. AntSeed CLI `0.1.201` separately resolves the published verifier
package at version `0.1.0`. Its flags select a verifier ID and require its
approval, without pinning a Git commit. The published package's ACI adapter,
quote checks, bindings, and verifier logic matched the deployed image when
checked on 2026-10-25.

1. Seller node (`seller-node-tee-genuine`, `seller-bound`): start an AntSeed
   buyer with:

   ```bash
   npx --package @antseed/cli@0.1.201 antseed buyer start \
     --peer 88c849029c02310ce360625a9d8e333a50f515d6 \
     --verifiers antseed-verifier \
     --require-verifier
   ```

   The node claim DCAP-verifies the seller CVM's TDX quote against Intel's
   root, checks acceptable TCB status and rejects debug mode, then checks the
   quote's binding to the buyer nonce and seller peer ID. `seller-bound`
   separately checks the seller's AntSeed identity signature over the nonce
   and the entire evidence bundle. Default acceptable TCB statuses are
   `UpToDate` and `SWHardeningNeeded`; `ANTSEED_VERIFIER_STRICT_TCB=true`
   restricts acceptance to `UpToDate`.
2. Gateway (`seller-provider-tee-genuine`): the seller's ACI adapter fetches a
   fresh report using the buyer's nonce. The verifier DCAP-verifies the
   gateway's TDX quote and checks its `report_data` commitment to that nonce
   and the reported keyset digest (`aci-keyset-v1`). It does not receive or
   recompute the full gateway keyset. This claim is informational by default:
   to require all three claims, set
   `ANTSEED_VERIFIER_REQUIRED_CAPS=seller-node-tee-genuine,seller-bound,seller-provider-tee-genuine`
   in the buyer's environment when starting it.
3. Independently: install `@phala/aci-verifier@0.8.0` in a separate directory
   and call its JavaScript API (this package has no `npx` executable):

   ```javascript
   import { verifyService } from '@phala/aci-verifier';

   const result = await verifyService('https://inference.phala.com');
   console.log(JSON.stringify(result, null, 2));
   ```

   It fetches `/v1/aci/attestation` with a fresh nonce and returns
   `{ verdict, lines, verification, composeHash }`. Live checks passed for
   the TDX quote (TCB `UpToDate`), recomputed keyset/nonce binding, keyset
   expiry, and the compose hash measured into RTMR3. The returned keyset
   lists domain-scoped TLS SPKI SHA-256 digests for `inference.phala.com`,
   `tee.redpill.ai`, and `api.redpill.ai`. These are public-key fingerprints,
   not certificates or public-key bytes. The plain call returns
   `verdict.verified: false`. Key custody (`id-5`) is skipped because this
   verifier does not implement the custody check. Only channel binding
   (`id-6`) fails, because no independently observed TLS SPKI was supplied.
   It does not check that the HTTPS connection used the attested key. The
   compose hash is measured, but is not approved by a release allow-list
   unless `acceptedComposeHashes` is supplied.

   Both `verifyService('https://inference.phala.com')` and
   `verifyService('https://tee.redpill.ai')` returned the same keyset digest
   and compose hash on 2026-10-25:

   ```text
   keyset: sha256:df2a7cd6c1c6b697879f64f758c2dc378d6c60cde8aa272e9c536858466383f5
   compose: aa767dbc886174737fcc77083851e3dbdea093b976404755a3f5a68c0d9e7992
   inference.phala.com TLS SPKI: 32a2825658a7377a37d6b6e854627dd16bdd8abd82353259884eac7866229ea7
   ```

   These are a verification snapshot, not permanent pins.
4. Seller CVM measurement (manual check): compare the seller app yourself
   using the
   [Phala Trust Center app](https://trust.phala.com/app/888a288a4ce0cb2b2debb03ea8eb750239656c78)
   page. It resolves to app `888a288a4ce0cb2b2debb03ea8eb750239656c78`, named
   `redpill-antseed-canary`. The verifier does not perform this comparison
   or enforce that app's measurements.

## What is NOT verified

- The verifier does not check the seller CVM's measurements against approved
  software: it never compares the seller's MRTD or RTMRs. With acceptable TCB,
  `seller-node-tee-genuine` passes for any genuine, non-debug TDX VM bound to
  the buyer nonce and this peer ID. The measured-image policy checks only
  the provider quote. Compare the seller app with the Trust Center page
  yourself.
- Without an enforced, buyer-approved measured-image policy,
  `seller-provider-tee-genuine` does not prove the quote came from the Phala
  gateway. The nonce-binding scheme and keyset digest are declared by the
  seller. Check that the claim detail says
  `bound to this round (aci-keyset-v1)`. If the seller omits the binding
  declaration, a genuine TDX quote with acceptable TCB and debug off still
  passes, with detail
  `freshness/instance binding NOT verified (no report_data scheme declared)`.
- Responses are not individually bound to an attested key; the current
  integration does not fetch or verify per-response ACI receipts for buyers.
- The seller's TLS connection to the gateway is not pinned to the attested
  keyset, so the attestations do not prove that your request took that path.
- The AntSeed ACI verifier does not recompute the gateway keyset or check
  source provenance. Its separate `seller-provider-measured-image` claim
  needs a buyer-supplied `ANTSEED_VERIFIER_MEASURED_IMAGE_POLICY`; to enforce
  it, also include that claim in `ANTSEED_VERIFIER_REQUIRED_CAPS`.
- `/v1/aci/attestation` supplies the gateway's own quote. The AntSeed ACI
  adapter relays that quote and digest, without upstream model-provider
  evidence or per-request serving evidence.
- A CPU TDX quote does not prove GPU execution. `is_tee` and capability
  metadata are declarations, not cryptographic proofs.

## Who sees plaintext

The local buyer proxy, seller CVM, Phala gateway, and serving upstream process
request content. The seller CVM and gateway can present TDX evidence, subject
to the path-binding limits above.

This peer advertises `transport.tcp-enc.v1`. AntSeed node `0.2.150` supports
encrypted TCP (a signed X25519 handshake with AES-256-GCM framing) and WebRTC
DTLS. It also retains a legacy plaintext TCP path, so an encryption capability
advertisement alone does not verify a particular buyer connection.

## Primary sources

- [Ownership proof](antseed.json)
  and `dig TXT _antseed.phala.com`.
- [Gateway model catalog](https://inference.phala.com/v1/models) and the
  seller's public discovery metadata at
  `http://tdx-prod13.phala.systems:14532/metadata`.
- [AntSeed verifier source](https://github.com/AntSeed/antseed-verifier/tree/8a94013c8746e114a2527483415b0eaf1ad23df2/src),
  especially `verifier.ts`, `caps/tee-tdx.ts`, `caps/seller-bound.ts`,
  `caps/measured-image.ts`, `adapters/aci.ts`, and `report-data.ts`.
- [AntSeed CLI 0.1.201](https://www.npmjs.com/package/@antseed/cli/v/0.1.201)
  (`buyer start --help`) and
  [node 0.2.150](https://www.npmjs.com/package/@antseed/node/v/0.2.150)
  (`dist/p2p/connection-manager.js` and `dist/p2p/secure-channel.js`), plus the
  [transport protocol](https://github.com/AntSeed/antseed/blob/main/docs/protocol/spec/02-transport.md).
- [ACI verifier 0.8.0](https://www.npmjs.com/package/@phala/aci-verifier/v/0.8.0),
  its `README.md`, `dist/transcript.js`, and the live API calls above.

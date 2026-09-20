---
generated: '2026-09-19'
method: generated
name: Verify an agent identity keylessly and check it against the ledger
description: Confirm that an IPv6 address or agent FQDN is a live Whisper agent, then prove its issuance sits in the signed transparency log and is not revoked — with no account, no key and no trust in the Whisper API.
api: openapi/whisper-online-openapi.json
operations: [verifyIdentity, ledgerCheckpoint, ledgerCheckpointKey, ledgerInclusion, ledgerStatusList, ledgerConsistency]
source: >-
  Grounded in openapi/whisper-online-openapi.json (operationIds verified verbatim) and
  https://whisper.online/docs/verify, /docs/transparency and /docs/a2a-trust. The first-resident
  address below is the one the provider publishes in llms.txt for exactly this purpose.
---

# Verify an agent identity keylessly

Use this before trusting an inbound A2A/MCP peer, or to check your own agent after `register`.

## Auth
- None. Every operation in this skill is keyless by design (see `authentication/whisper-online-authentication.yml`).

## Steps
1. **Ask for the verdict** — `verifyIdentity` (`GET /verify-identity?address=<addr-or-fqdn>`; the docs and CLI use the same handler at `https://rdap.whisper.online/verify-identity/<addr>`). Read `is_whisper_agent`, `dane_ok`, `jws_ok`, `fqdn`, `tenant` and the `evidence` block (ptr, forward_aaaa, dane_tlsa_sha256, rdap, identity_doc). A non-agent address returns a clean `{"is_whisper_agent": false}`, never an error; a missing address is a 400.
2. **Cross-check with stock tools if you can** — `dig -x <addr> +short` must return the `fqdn`; `dig +short AAAA <fqdn>` must return the address; `dig +short TLSA _443._tcp.<fqdn>` must be a `3 1 1` record matching `dane_tlsa_sha256`. Require the `AD` flag (DNSSEC-validated).
3. **Fetch the signed checkpoint** — `ledgerCheckpoint` (`GET /checkpoint`, text/plain C2SP note: origin `whisper.online/ledger/g2`, tree size, root hash, Ed25519 signature and any witness cosignatures) and `ledgerCheckpointKey` (`GET /checkpoint/key`). Pin the key from DNS instead of this fetch when possible: `dig +short TXT _whisper-ledger.whisper.online`.
4. **Prove inclusion** — `ledgerInclusion` (`GET /inclusion?leaf=<N>` where N is a leaf index or a 64-hex statement digest) returns `leaf_hash`, the sibling `proof[]` and the checkpoint it folds to. Fold the path to the root and compare with step 3. One agent's own events and proof are also at `https://rdap.whisper.online/ip/<addr>/transparency`.
5. **Check it is not revoked** — `ledgerStatusList` (`GET /checkpoint/status-list`) is the signed revocation status-list; a revoked identity also answers `is_whisper_agent: false` in step 1 and `dig -x` returns nothing within ~60 seconds (60-second TTLs).
6. **Optionally prove append-only history** — `ledgerConsistency` (`GET /consistency?old=<a>&new=<b>`) proves an earlier tree is a prefix of a later one.

## Errors
- `verifyIdentity` 400 `{"error":"bad_request","detail":"provide an IP literal or an agent FQDN ..."}` when no address is given.
- `ledgerInclusion` 400 on a missing or malformed leaf ("a bad leaf is a clear 400, never a 500").
- Read `X-Whisper-Ledger-Claim` on ledger responses for the current transparency claim level. See `errors/whisper-online-problem-types.yml`.

## Notes
- Try it against the published first resident: `2a04:2a01:b69a:6717:e3b0:51ff:3bf7:f478` (fqdn `ae3b051ff3bf7f478.tdc38e7c55bad3306a92b830f9bb1e4f9.agents.whisper.online`).
- Verify is not authorize: a passing identity tells you WHO the peer is, not what it may do (docs/a2a-trust). Gate by exact name, tenant subdomain or wildcard after verifying.
- The CLI's `whisper verify --trustless <addr>` re-derives all of the above from the IANA DNSSEC root with the Whisper API untrusted.

---
generated: '2026-09-19'
method: generated
name: Assess a destination before connecting to it
description: Ask the Whisper graph who really runs a host and whether it is safe, with the evidence chain behind the answer, before an agent opens a connection — keyless.
api: openapi/whisper-online-openapi.json
operations: [query]
source: >-
  Grounded in openapi/whisper-online-openapi.json (operationId query, POST /api/query) and
  https://whisper.online/docs/graph-api. Verified live 2026-09-19: a keyless
  CALL whisper.identify("api.openai.com") returned vendor Cloudflare / category cdn with its evidence chain.
---

# Assess a destination before connecting

## Auth
- None for the reads below. Send `X-API-Key` only if you also need `CALL whisper.agents` or `submit`. From inside a connected agent you can skip the key entirely and call `https://[<your /128>]/identify?q=<host>` — the address is the credential.

## Conventions
- One endpoint: `query` = `POST https://graph.whisper.online/api/query` (the spec's `servers[]` is `https://whisper.online`; the docs name `graph.whisper.online`). JSON body `{"query": "...", "parameters": {...}}`. Always bind values as `$parameters`; never concatenate them into the Cypher.
- Keyless calls are capped at 100 rows per query. Envelope is `{columns, rows, statistics}`; errors are RFC 7807 `application/problem+json` — branch on the `type` slug (see `errors/whisper-online-problem-types.yml`).

## Steps
1. **Who operates it?** — `query` with `CALL whisper.identify($h)` and `{"parameters":{"h":"api.example.com"}}`. Read `canonical_name`, `category`, `roles`, `confidence`, `band` (DIRECT > DERIVED > HEURISTIC > UNKNOWN) and `evidence[]` — the literal traversal (e.g. `RESOLVES_TO->IPV4->DELEGATED_TO->VENDOR:cloudflare`).
2. **Is it safe?** — `query` with `CALL whisper.assess($targets)` and `{"parameters":{"targets":["185.220.101.1"]}}`. Read `band`, `verdictScore`, `isThreat`. Read `coverage` before `band`: `level NONE` means not listed at this granularity, `band UNKNOWN` means never seen — neither means safe.
3. **Why?** — `query` with `CALL whisper.explain($t)` for the feeds that fired, the score and the age of the freshest observation. Note `available:false` + `retryAfter` when the threat-intel backend is briefly unavailable (HTTP still 200) — wait and retry.
4. **Decide, then optionally enforce** — with a key, `CALL whisper.agents({op:'policy', args:{default:'deny', allow:[...]}})` makes the resolver refuse everything you did not allow; no key is needed to observe a deny (`dig @<your resolver /128> <name> AAAA +dnssec` returns NXDOMAIN).

## Errors
- 400 `query-error` (malformed Cypher / unbound `$parameter`), 400 `query-depth-exceeded` (keyless depth; add a key or stage the traversal), 408 `query-timeout`, 429 `query-quota-exceeded` (wait; batch indicators into one `UNWIND`).
- Procedure names are matched case-insensitively and a near-miss can resolve to something unrelated — confirm with `CALL db.procedures()` (`psl.tldPlusOne`, not `psl`).

## Notes
- Cognition calls target under 300 ms; the provider's own resolver treats a slow answer as "no opinion" and fails open — design your gate the same way.
- Trial keys: 10 requests/minute, 500/day (`rate-limits/whisper-online-rate-limits.yml`).

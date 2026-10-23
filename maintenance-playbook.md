# Maintenance Playbook

FTP will maintain the Canton x402 stack after Milestone 1.3 as open-source ecosystem infrastructure. Maintenance covers the Canton x402 facilitator, client SDK, resource server middleware, scheme specification, deployment documentation, and CanTrustAI reference implementation.

## 1. Maintenance Scope

Maintenance includes compatibility and operational updates required to keep Canton x402 usable as Canton, the token standard, and upstream x402 evolve.

FTP will maintain:

- compatibility with supported Canton Ledger API versions;
- compatibility with Canton Coin payment flows;
- compatibility with supported CIP-56 token flows;
- updates required by Token Standard V2 / CIP-112 as those changes become available in Canton tooling;
- updates required by upstream x402 specification or facilitator interface changes;
- updates to the Canton x402 payment scheme specification;
- fixes for facilitator /verify


and keep builder-facing documentation current, including:

- facilitator deployment guide;
- SDK quickstart;
- middleware quickstart;
- Canton x402 payment-flow explanation;
- compatibility matrix;
- troubleshooting guide;
- security and audit status;
- examples for Canton Coin and supported CIP-56 / CIP-112 token flows.

## 2. Health checks

The facilitator serves `/ready` and `/metrics` on its own port (4022 by default). Keep both off the public edge; the hosted facilitator answers 404 for them by design. Read them on the host or over a private link:

```bash
docker compose exec facilitator wget -qO- http://127.0.0.1:4022/ready
docker compose exec facilitator wget -qO- http://127.0.0.1:4022/metrics
```

- `/ready` returns 200 when every dependency answers, and 503 with the failing check named otherwise. Checks: `token` (ledger auth), `participant`, `secondary_participant` (only when a second participant is configured) and `scan`.
- `/health` is a static liveness answer and `/supported` is static config. Neither proves that the participant or Scan is reachable.
- `/metrics` is Prometheus. Useful series: `x402_facilitator_settle_total`, `x402_facilitator_settle_duration_seconds`, `x402_facilitator_breaker_open_total`, `x402_facilitator_inline_key_lookup_total` (payer key lookups; `unavailable` means the topology reader is down).
- The real gate after any change is one live paid request to a 402 resource you control. Confirm the settlement `updateId` on the ledger or Scan. A success response alone is not proof.

## 3. Upgrade and rollback

The facilitator runs as a container. Its version is the image tag in the compose project's `.env`.

1. Back up `.env` and the compose file.
2. Build the new image with a new tag and set `IMAGE_TAG` in `.env`.
3. Recreate only the facilitator:
   ```bash
   docker compose up -d --no-deps --force-recreate facilitator
   ```
4. Deploy in two phases: first the new image with the old settings and a live check, then any new settings. This keeps a code regression apart from a config change.
5. Run the health checks above and one live payment.

Rollback: set the previous `IMAGE_TAG` and recreate the container the same way. Keep old images on the host. Postgres state survives image swaps.

Notes:

- Do not overwrite the host's `.env` or compose file with repository copies. Diff them. Host-only entries (for example Scan host mappings) are easy to lose and break every payment while `/health` stays green.
- A variable in `.env` reaches the container only if the compose file passes it in `environment:`. `docker restart` does not reread env; recreate the container.

## 4. Participant upgrade

A participant upgrade is the highest-risk change.

1. Read the Canton and Splice release notes for Ledger API and Scan changes.
2. Rehearse on a test network first. Avoid MainNet topology freeze and logical synchronizer upgrade windows.
3. Upgrade the facilitator image first if the new participant needs it (see section 5).
4. The Splice validator `start.sh` recreates its docker network and fails if containers from other compose projects are attached. Disconnect them first (for example a participant bridge or the topology reader), then reconnect and restart them after the upgrade.
5. Check that the topology reader still returns payer keys. Without it every inline payment is refused as `invalid_exact_canton_signature_invalid`.
6. Check package vetting (section 6).
7. Run one live payment after each step.

With two participants: keep package vetting identical on both, and keep a retired participant configured as secondary for at least 600 seconds (the longest `executeBefore`) after its last settle.

## 5. Canton 3.6 and Splice 0.9

- Canton 3.6 disables `POST /v2/updates/transaction-by-id`; 3.7 removes it. Use `update-by-id`.
- Canton 3.6 rejects the top-level `filter` and `verbose` fields on `/v2/state/active-contracts`; 3.7 removes them. Send the filter inside `eventFormat`.
- Splice 0.9.0 removed Scan v0/v1 `holdings/state` and `/state/acs`. Use the v2 endpoints.
- `/v2/jose/jwks` is not in Splice-shipped participants. The facilitator gets payer signing keys from the topology reader, which reads the participant admin API.
- Participants cap list responses (commonly 200 elements). Read large ACS sets in bounded slices with `?limit=N`.
- Anything that talks to a 3.6 participant directly needs `@ftptech/x402-canton-ledger` 0.6.2 or later. Deploy a facilitator built on it before the participant moves to 3.6. Wallets and the MCP talk only to the relay and are not affected.

## 6. DAR duty

The payment path uses no custom DAR. The duty is to keep third-party packages vetted identically on every participant that hosts facilitator, agent or merchant parties:

- Splice release DARs.
- The DA Registry Utility bundle, for CIP-56 registry tokens.
- Token standard v1 and v2 interfaces.

Compare vetted packages across participants before a cutover and after every DAR upload. Before a network package-set change, confirm the participant has every DAR it needs; a missing one can put the participant into a crash loop.

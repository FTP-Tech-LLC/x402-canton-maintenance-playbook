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

# Mainbrella sandbox platform roadmap

Updated October 6, 2026.

Mainbrella provides hosted Linux sandboxes for developer and agent workloads. The
product is the account-facing sandbox API, JavaScript and Python clients, web
dashboard, and iPhone/iPad client. The earlier AWS/GCP inventory-manager concept
is no longer the product direction.

## Release status

- Container lifecycle, five machine sizes, HTTP and managed execution,
  reconnectable output, binary files, filesystem operations, stdin/signals/PTY,
  browser terminal, SSH, API keys, image catalog, custom images, and OpenAPI exist.
  Check deployed capabilities rather than assuming every feature is enabled.
- Protected web previews are live and qualified, including Next.js assets,
  WebSockets/HMR, account isolation, generation replacement, and stop revocation.
- Saved filesystem workspaces are enabled in production. The two-start customer
  API qualification passed its eight behavior checks; the initial verifier
  cleanup rejection was reconciled with authenticated generation-absence checks.
  Save explicitly before stop; restore creates a fresh container and preserves
  files, not memory or running processes. Quotas, retention, and image
  compatibility apply. Live dashboard and installed Python save/restore remain
  separate verification gaps. Evidence and rollout details live in
  `backend/docs/workspaces.md` and its private qualification records.
- Python SDK 0.1.0 is published on PyPI; fresh wheel/source downloads match the
  qualified hashes and both clean installations pass their contract suites.
  JavaScript 0.1.0 passes artifact and deployed qualification but remains
  staged on npm as `@mainbrella/sdk` 0.1.0, with a verified archive checksum,
  awaiting registry validation and browser 2FA approval. A checksum-pinned
  GitHub OIDC staging workflow is prepared but not yet configured in npm.
  The deployed SDK gate used two total starts; Python’s default HTTP user agent
  was fixed before its successful remaining start. Follow `backend/docs/sdk-release.md`.
- Internet-off mode exists but remains disabled pending its complete release gate.
  Metrics and webhooks remain unqualified; lifecycle history is available.

## Work in order

1. Keep backend and web CI green, including deterministic portable skill archives.
2. Complete dashboard and installed SDK workspace save/restore/export verification
   within explicit start budgets; keep independent exports and recovery evidence.
3. Publish the exact qualified JavaScript SDK 0.1.0 artifact to npm after resolving
   browser stage approval. Python is released. Replace JavaScript archive-install
   examples only after registry downloads are verified.
4. Finish the internet-off release gate before enabling it.
5. Build secret mediation and exact-destination egress policy: credentials stay in
   a trusted mediator outside guest environment variables and process arguments.
6. Add ergonomic snapshot/fork fan-out around saved workspaces. Benchmark the time
   from fork call to first successful execution; do not add a second storage system.
7. Replay the corrected bounded benchmark matrix, then consider mobile approval
   notifications when agent demand is established.

Suspend/resume that preserves process state should follow demonstrated customer
need. Defer major iOS expansion, additional preview work, marketing-page expansion,
metrics/webhooks/OTLP qualification, and regions while the release gates above
remain open. Do not describe unqualified or unpublished capabilities as shipped.

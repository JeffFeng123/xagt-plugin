# Agentonomy Commerce

> Submission: Agentonomy has supplied its identity and support contact. The submitter authorized publication of the nonsensitive code and materials after personal sample identifiers were replaced. Public HTTPS deployment and purchase/restart verification passed.

## Capability

- **One-line description:** Let an Agent purchase a service within a user-authorized budget and reliably retrieve its result; this review demonstrates paid CSV reconciliation.
- **Who it helps:** Agents operating a user-authorized commerce budget.
- **Capability boundary:** The review runtime validates identity, policy, budget, payment evidence, and merchant delivery. The merchant call is real loopback HTTP: the signed logical resource `https://merchant.agentonomy.invalid/v1/reconcile` is routed to a local `http://127.0.0.1:<port>/v1/reconcile` listener. Chain settlement is simulated, `real_funds` remains false, and no real funds are spent.

## Live API

- **API base URL:** https://review.agentonomy.xyz
- **Health-check URL:** https://review.agentonomy.xyz/health
- **Authentication:** Bearer token required; contact fengjie@alvinsclub.ai to arrange reviewer access through an approved private channel. No token is committed, and access handoff is pending.
- **Rate limits / known limits:** 60 authenticated requests per minute by default; 256 KiB maximum HTTP body; 128 KiB maximum UTF-8 CSV; at most 1,000 rows including duplicates; one three-letter currency per CSV; each amount must be a two-place decimal with absolute value at most 1,000,000,000,000; 0.30 sandbox USDC per delivered report; 1.00 sandbox USDC initial budget; signed bootstrap grant valid for 30 days; previews expire after 300 seconds; result payloads are retained for 7 days; the SQLite review state and settlement journal persist across restart.
- **API contract:** `GET /health` and `GET /.well-known/xagent-verification.json` are public. Bearer-authenticated `GET /v1/services`, `GET /v1/budget`, `POST /v1/previews` (JSON `{"offering_id":"csv-reconciliation-v1","csv_text":"..."}` plus an `Idempotency-Key` header), `POST /v1/purchases` (JSON `{"preview_id":"..."}`), and `GET /v1/purchases/{purchase_id}` are implemented in `source/`.

## Source and reproducibility

- **Source repository:** https://github.com/HEchooo/agentonomy-commerce
- **Review commit:** `79b657e7c63158fafc657cf6370048bcde384c57`
- **Complete review source:** `source/`
- **Run tests:** `make PYTHON=.venv/bin/python test-commerce test-review test-submission`
- **Run locally:** `make PYTHON=.venv/bin/python demo` (simulated settlement only)
- **Deploy:** See [deployment instructions](source/docs/deployment.md). The independent GCP deployment is live at https://review.agentonomy.xyz; see verification evidence below.
- **Version binding:** The deployed health endpoint and `/.well-known/xagent-verification.json` must expose this exact commit before review.

The public API was checked and exposes:

```json
// GET https://review.agentonomy.xyz/health
{"status":"ok","commit":"79b657e7c63158fafc657cf6370048bcde384c57"}
```
```json
// GET https://review.agentonomy.xyz/.well-known/xagent-verification.json
{"schemaVersion":1,"slug":"hechooo-agentonomy-commerce","commit":"79b657e7c63158fafc657cf6370048bcde384c57"}
```

## Source integrity and historical metadata

The authoritative review snapshot is the outer `source-manifest.json` and `source-manifest.sha256`: all 733 files match the declared review commit. `source/docs/source-manifest.json` is an inherited **historical Clink working-tree export**, not the manifest for this review version. Its 678 entries predate later changes (36 recorded hashes differ and 55 current files are absent). It is preserved as part of the exact source commit and must not be used to validate this submission. The outer manifest covers the current complete snapshot.

## Verification

Repeatable health, quote, purchase, read, replay, and safe-failure calls are
documented in `verification/README.md`. Original purchase/delivery acceptance passed at the prior commit. Current-source public health/proof, original-order replay after a VM stop/start and source upgrade, container restart and authenticated HTTPS replay have passed; see `verification/upgrade-evidence.json`. Review access still needs a private handoff. The checked verifier is
`source/scripts/verify_review_api.py`:

```bash
BASE_URL='https://review.agentonomy.xyz'
export AGENTONOMY_API_TOKEN='<review-token>'
.venv/bin/python source/scripts/verify_review_api.py \
  --base-url "$BASE_URL" \
  --expected-commit "79b657e7c63158fafc657cf6370048bcde384c57" \
  --purchase-id "$PURCHASE_ID" \
  --preview-id "$PREVIEW_ID"
```

Use IDs returned by the walkthrough or provided through the private review handoff. Running the verifier without IDs is only appropriate for a fresh local sandbox, because it checks a new 0.30 charge. The live sandbox already contains the deployment acceptance purchase.

- **Health-check result:** PASS over public HTTPS, exact commit match; see `verification/public-https-evidence.json`.
- **Capability call:** PASS: synthetic CSV produces two unique transactions, duplicate `t1`, USD net `27.50`; used budget 0.30, remaining 0.70, one settlement and one delivery. VM stop/start and source upgrade, container restart and authenticated HTTPS replay preserved the same order and counters; see `verification/upgrade-evidence.json`.
- **Expected error behavior:** Invalid authentication, malformed input, limits, expired authorization, and unavailable merchant fail closed without retrying a settled payment.

## Security and data handling

- **Data collected:** Review purchase inputs, bounded CSV contents, authorization state, and redacted result metadata.
- **Purpose and retention:** Preview/input records expire after 300 seconds; delivered result payloads are retained for 7 days; persistent SQLite review state and the simulated settlement journal survive restart. Credentials and raw secrets are not logged.
- **Third parties / outbound network calls:** The merchant uses an internal logical `.invalid` resource routed to a fixed loopback HTTP listener. Core and Marketplace run in the local composition; tests make no production outbound calls.
- **Secrets:** No secrets are committed. Review access is supplied only through an approved private channel when required.
- **Known risks / restrictions:** Settlement is explicitly simulated and is not evidence of a live blockchain transaction. The deployed sandbox is shared and budget-limited. API credentials currently require manual rotation; the 30-day spending grant is separate from token expiry. Reviewer access has not yet been handed off. No formal security audit or real-funds test is claimed.

## Support

- **Team / builder:** Agentonomy
- **Contact:** fengjie@alvinsclub.ai
- **License / rights:** See [RIGHTS.md](RIGHTS.md); the declaration applies to this sanitized submission. Third-party terms remain applicable; no blanket relicensing is asserted.

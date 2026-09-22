# Verification evidence

Public deployment and cloud acceptance are complete. Submitter: Agentonomy; support: fengjie@alvinsclub.ai. Private reviewer access remains pending; source publication and official archive authorization apply to this sanitized artifact. See `public-https-evidence.json` and `upgrade-evidence.json`.

## Prerequisites

- Review commit: `79b657e7c63158fafc657cf6370048bcde384c57`
- API base URL: `https://review.agentonomy.xyz`
- Authentication: request review access from fengjie@alvinsclub.ai through an approved private channel. Existing server credentials require manual rotation; no token is committed and reviewer handoff is pending.

## 1. Health check

```bash
curl --fail --silent --show-error https://review.agentonomy.xyz/health
```

Expected response:

```json
{"status":"ok","commit":"79b657e7c63158fafc657cf6370048bcde384c57"}
```

## 2. Deployment proof

```bash
curl --fail --silent --show-error https://review.agentonomy.xyz/.well-known/xagent-verification.json
```

Expected response:

```json
{"schemaVersion":1,"slug":"hechooo-agentonomy-commerce","commit":"79b657e7c63158fafc657cf6370048bcde384c57"}
```

## 3. Reproducible API walkthrough

Set `BASE_URL` to the deployed root origin and provide the review
token through the environment. The API base is the origin; `/v1/` is the
capability path.

```bash
BASE_URL='https://review.agentonomy.xyz'
export AGENTONOMY_API_TOKEN='<review-token>'
CSV='transaction_id,date,description,amount,currency,category
t1,2026-09-01,Hosting,-12.50,USD,software
t2,2026-09-02,Invoice,40.00,USD,revenue
t1,2026-09-01,Hosting,-12.50,USD,software'

curl --fail --silent --show-error "$BASE_URL/v1/services" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN"
curl --fail --silent --show-error "$BASE_URL/v1/budget" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN"

PREVIEW=$(curl --fail --silent --show-error \
  --request POST "$BASE_URL/v1/previews" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN" \
  --header "content-type: application/json" \
  --header "Idempotency-Key: review-preview-1" \
  --data "$(python3 -c 'import json,sys; print(json.dumps(dict(offering_id="csv-reconciliation-v1", csv_text=sys.argv[1])))' "$CSV")")
PREVIEW_ID=$(python3 -c 'import json,sys; print(json.load(sys.stdin)["preview_id"])' <<<"$PREVIEW")

PURCHASE=$(curl --fail --silent --show-error \
  --request POST "$BASE_URL/v1/purchases" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN" \
  --header "content-type: application/json" \
  --data "$(python3 -c 'import json,sys; print(json.dumps(dict(preview_id=sys.argv[1])))' "$PREVIEW_ID")")
PURCHASE_ID=$(python3 -c 'import json,sys; print(json.load(sys.stdin)["purchase_id"])' <<<"$PURCHASE")
curl --fail --silent --show-error "$BASE_URL/v1/purchases/$PURCHASE_ID" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN"

# Replay the same preview key and verify the same purchase/result is returned.
REPLAY=$(curl --fail --silent --show-error \
  --request POST "$BASE_URL/v1/previews" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN" \
  --header "content-type: application/json" \
  --header "Idempotency-Key: review-preview-1" \
  --data "$(python3 -c 'import json,sys; print(json.dumps(dict(offering_id="csv-reconciliation-v1", csv_text=sys.argv[1])))' "$CSV")")
REPLAY_PURCHASE=$(curl --fail --silent --show-error \
  --request POST "$BASE_URL/v1/purchases" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN" \
  --header "content-type: application/json" \
  --data "$(python3 -c 'import json,sys; print(json.dumps(dict(preview_id=sys.argv[1])))' "$PREVIEW_ID")")
curl --fail --silent --show-error \
  "$BASE_URL/v1/purchases/$PURCHASE_ID" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN"
REPLAY_BUDGET=$(curl --fail --silent --show-error \
  "$BASE_URL/v1/budget" \
  --header "authorization: Bearer $AGENTONOMY_API_TOKEN")
```

The preview response identifies `csv-reconciliation-v1` and the 0.30 sandbox
USDC price. The purchase response must report `state: delivered`,
`service_transport: http`, `real_funds: false`, and
`settlement_mode: simulated`; the sample CSV has two unique transactions,
duplicate ID `t1`, and a USD net total of `27.50`. A replay must preserve the
purchase/result and leave the budget unchanged relative to the immediately preceding purchase. The deployment acceptance order already consumed 0.30, so a separate new walkthrough purchase increases total used budget to 0.60 rather than resetting it to 0.30. The final `REPLAY_BUDGET` response is the value to
compare with the budget response immediately after the first purchase.
The loopback merchant is HTTP, but the simulated settlement is not a live
blockchain transaction.

The same verifier used for review can run this sequence and check the commit:

```bash
export AGENTONOMY_API_TOKEN='<review-token>'
.venv/bin/python source/scripts/verify_review_api.py \
  --base-url "$BASE_URL" \
  --expected-commit "79b657e7c63158fafc657cf6370048bcde384c57" \
  --purchase-id "$PURCHASE_ID" \
  --preview-id "$PREVIEW_ID"
```

## 4. Safe failure checks

```bash
curl --silent --show-error \
  --request POST "$BASE_URL/v1/previews" \
  --header "authorization: Bearer <invalid-token>" \
  --header "content-type: application/json" \
  --header "Idempotency-Key: invalid-auth" \
  --data '{"offering_id":"csv-reconciliation-v1","csv_text":"bad"}'
```

Expected failures include 401 for invalid authentication, 422 for malformed
input or a missing/invalid `Idempotency-Key`, 413 for a body over 256 KiB, and
400 for a CSV over 128 KiB, 1,000 rows, mixed currencies, or an amount whose
absolute value exceeds 1,000,000,000,000.
Expired authorization and unavailable merchant conditions fail closed without
retrying a settled payment.

## Existing live sandbox

At acceptance and after the source upgrade, used budget is 0.30 and remaining budget is 0.70 simulated USDC. Reuse returned purchase/preview IDs for repeated verification. Do not reset state, grant budget or create another payment to recover a completed payment. API tokens and saved order IDs can be handed to reviewers privately; neither is included in these public evidence files. Results expire after seven days and grants after thirty days.

## Source integrity

Use the outer `../source-manifest.json` and `../source-manifest.sha256` for the 733-file review snapshot. `../source/docs/source-manifest.json` is inherited historical metadata and does not describe this current snapshot. See the source-integrity note in `../SUBMISSION.md`.

## Current version and original acceptance

The current source commit is `79b657e7c63158fafc657cf6370048bcde384c57`. `upgrade-evidence.json` verifies the same original order and counters after VM stop/start, source upgrade, container restart and authenticated HTTPS replay. `history/a5f0e4f/` records the original first purchase and its earlier restart checks at the prior commit; these historical files are not represented as checks of the current commit. The authoritative current test results are in `TEST_RESULTS.md`.

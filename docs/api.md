# Credit-risk scoring API

Team: 2

Local base URL: `http://127.0.0.1:8000` after `python run.py serve`.
Interactive documentation: `http://127.0.0.1:8000/docs`.
Cloud deployment and the live base URL have not yet been verified. Record the verified
Cloud Run URL in `docs/deployment.md` after deployment.

## Endpoints

The examples below use the locally exported, verified model. Set `BASE_URL` to the local
address above, or to the verified Cloud Run address after deployment.

### GET /health

Reports whether startup completed and a model is loaded. It does not score an applicant.

```bash
curl "$BASE_URL/health"
```

```json
{"status":"ok","model_loaded":true,"model_version":"1"}
```

### GET /model-info

Identifies the serving model and its decision threshold so a response can be traced to a
specific model and feature snapshot.

```bash
curl "$BASE_URL/model-info"
```

```json
{"name":"credit-risk","version":"1","threshold":0.085,"n_features":164,"valid_auc":0.735588871424925,"data_version":"c393a114d6e3","exported":"2026-11-13T19:03:41+00:00"}
```

### POST /predict

Scores one applicant in the 5,000-row feature store. Supply the positive integer
`SK_ID_CURR`. Optional `AMT_INCOME_TOTAL`, `AMT_CREDIT`, and `AMT_ANNUITY` must be positive
numbers when supplied; omitted or null amounts retain their stored values. Amount changes
are applied to a copy of the row and the dependent ratios are recomputed. No history is
created for an ID outside the snapshot.

```bash
curl -X POST "$BASE_URL/predict" -H 'Content-Type: application/json' \
  -d '{"SK_ID_CURR":100021}'
```

```json
{"SK_ID_CURR":100021,"probability":0.10431654676258995,"decision":"reject","threshold":0.085,"model_version":"1"}
```

The probability estimates default risk. A probability greater than or equal to 0.085
returns `reject`; a lower probability returns `approve`. These are course demonstration
outputs from a fixed dataset.

## Errors

HTTP 422 means the request does not satisfy the schema. A missing or invalid ID, an amount
of zero or less, text instead of a numeric amount, or an unexpected field is rejected before
scoring. For example:

```bash
curl -X POST "$BASE_URL/predict" -H 'Content-Type: application/json' \
  -d '{"SK_ID_CURR":100021,"AMT_INCOME_TOTAL":-1}'
```

```json
{"detail":[{"type":"greater_than","loc":["body","AMT_INCOME_TOTAL"],"msg":"Input should be greater than 0","input":-1,"ctx":{"gt":0.0}}]}
```

HTTP 404 means the request is valid but the applicant is absent from the store:

```bash
curl -X POST "$BASE_URL/predict" -H 'Content-Type: application/json' \
  -d '{"SK_ID_CURR":1}'
```

```json
{"detail":"Applicant 1 is not in the feature store"}
```

## Response times

A local HTTP measurement on November 13, 2026 used one first request followed by 100 scored
requests, with zero HTTP errors. `reports/latency_local.json` records p50 **9.61 ms**, p95
**10.32 ms**, maximum **14.47 ms**, and mean **9.73 ms**. The first measured request took
**62.5 ms**. Model loading had already completed, so this first request is not a measured
Cloud Run cold start. The local server used `127.0.0.1:8045` to avoid conflicts with other
development services. These timings exclude network travel to Google Cloud.

p50 is the median wait: half of successful measured requests were at or below that time.
p95 is the 95th percentile: roughly five percent were slower. p95 therefore helps describe
the longer delays an applicant might see on a loan screen. The required live service
measurement remains to be taken after deployment with
`python run.py latency --url <verified service URL>`; that command will write
`reports/latency.json` for the submission. A first request measures a cold start only when
no instance is already running.

## Online or batch scoring?

Online scoring answers one applicant when a form is submitted. It suits an interactive
credit application because the applicant can receive a response while still on the screen.
It needs a continuously reachable service, predictable latency, and clear failure handling.
Scaling to zero reduces idle resource use but introduces startup delay after quiet periods.
Batch scoring processes many applicants on a schedule and can share startup and processing
costs across the whole group. It is useful for periodic portfolio monitoring or refreshing
stored scores when an immediate answer is unnecessary. Its tradeoff is that a score can be
out of date between runs. For this course's single-application decision, online scoring
fits the interaction, while a scheduled batch pipeline can refresh the feature snapshot.

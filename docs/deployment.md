# Deployment (Week 5)

Team: 2

Status: local implementation, model export, and API checks are verified. Cloud deployment
has not been performed or verified from this working copy; no live service URL is claimed.

| Setting | Prepared value or observed status |
|---|---|
| Service URL | Awaiting verified Cloud Run deployment |
| Project id / region | `credit-risk-team2` / `us-central1` |
| Image | `us-central1-docker.pkg.dev/credit-risk-team2/credit-risk/api:week5` (prepared; not built or pushed) |
| Model bucket | `gs://credit-risk-models-team2` (proposed private bucket; availability and configuration require cloud verification) |
| Model version | Locally exported version `1`; threshold `0.085`; 164 features; data version `c393a114d6e3` |
| Instances | Prepared minimum 0, maximum 2 |
| Memory / CPU | Prepared 1 GiB / 1 CPU; concurrency 10 |
| Service account | `credit-risk-api@credit-risk-team2.iam.gserviceaccount.com`; prepared bucket-scoped Storage Object Viewer role |

## Verified local model lineage

The current remote main at `429c09b4b615a985774b58fcc8bd47955be60d79` records a calibrated
histogram gradient boosting model with threshold 0.085 and validation AUC
0.735588871424925. Its original serialized artifact was unavailable locally. The same
specification, calibration, seed, fixed split and feature table were used to rebuild it in a
fresh local registry. Validation AUC, Brier score, mean predicted probability, expected cost
and rejection rate reproduce the recorded values exactly. `reports/final_model.json`
records both the original source run and the new local run under `artifact_rebuild`.
The different older model with threshold 0.100 and AUC 0.752589368476202 was not used.

## Deployment procedure

Review `scripts/deploy_week5_cloudshell.sh` and verify the intended project, bucket name,
permissions and service settings. In Cloud Shell, use the reviewed source branch and place
the three exported files in `models/`. The script creates or reuses a private bucket,
uploads that bundle, gives the service account read access to that bucket, builds the
image, and deploys the public course API with the settings above. Public access prevention
stays enabled on the model bucket. No account key is required or committed.

After deployment, record the actual service URL, deployed image digest, service account and
observed settings in this file. Confirm `/health`, `/model-info`, a known prediction, a 422
response and a 404 response. Measure the live URL with `python run.py latency --url
<verified service URL>`, retain its output in `reports/latency.json`, and run the Wednesday
and full Week 5 checks. Local timings in `reports/latency_local.json` do not satisfy that
cloud requirement.

## Deploying a new model

Register and verify the new model locally, then run `python run.py export`. Check that all
three bundle files agree on version, features, threshold and data fingerprint. Upload all
three together to the private bucket and redeploy a new Cloud Run revision so each instance
loads the same snapshot. Verify the live `/model-info` and a prediction against the local
bundle before calling the rollout complete. Repeat the live latency measurement and Week 5
checks, then update the deployment record. Keep the service available through the end of
the course as required.

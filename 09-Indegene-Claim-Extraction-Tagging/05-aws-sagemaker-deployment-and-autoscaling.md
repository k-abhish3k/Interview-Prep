# 05 — AWS Sagemaker Deployment and Auto-Scaling

## Why this chapter matters

This chapter maps directly onto the back half of the resume bullet: "Implemented Auto-scaling to
facilitate load balancing for single/multi model endpoints... Used AWS Sagemaker to fine-tune
associated models, S3 to store model artifacts, Eventbridge Scheduler to perform health checks,
Lambda, and API gateway for model deployment." Each of those is a distinct AWS service doing a
distinct job in the MLOps diagram from chapter 00. Knowing what each one is *for* — not just its
name — is what separates a candidate who can defend this bullet from one who's just listing acronyms.

## Sagemaker fine-tuning jobs

AWS Sagemaker is a managed ML platform; the piece relevant here is **Sagemaker Training Jobs**, which
takes a training script (the fine-tuning code from chapter 02), a specified instance type, and
pointers to training data in S3, and runs it on managed, ephemeral compute — you don't provision or
maintain the GPU instance yourself, you submit a job and Sagemaker handles provisioning, running the
script, and tearing the compute down afterward.

```python
from sagemaker.huggingface import HuggingFace

huggingface_estimator = HuggingFace(
    entry_point="train.py",              # the fine-tuning script from chapter 02
    source_dir="./scripts",
    instance_type="ml.g4dn.xlarge",       # GPU instance, provisioned only for the run
    instance_count=1,
    role=sagemaker_execution_role,
    transformers_version="4.x",
    pytorch_version="2.x",
    py_version="py310",
    hyperparameters={"epochs": 3, "learning_rate": 2e-5, "model_name": "bert-base-uncased"},
)

huggingface_estimator.fit({"train": "s3://claims-bucket/training-data/v17/"})
```

Why this matters over just running the script on a persistent server: training jobs are bursty (you
need a GPU for the hours a fine-tuning run takes, not 24/7), so paying for ephemeral managed compute
that spins up for the job and shuts down after is materially cheaper than idling a GPU instance
between retraining cycles — and it's what makes the "automated retraining pipeline" from chapter 04
practical to run on a schedule without a human provisioning infrastructure each time.

## S3 as the artifact store

Every Sagemaker training job writes its output — model weights, tokenizer config, any training
metadata — to S3 as the artifact store (`model.tar.gz` at a versioned path). This is the "maintenance
of associated model artifacts" from the resume bullet, made concrete: S3 gives you durable,
versioned, cheap storage that's decoupled from any single compute instance, which is exactly the
property you need for the artifact-lineage practice described in chapter 04 (never overwrite, always
version, always traceable). A typical layout:

```
s3://claims-bucket/model-artifacts/
  claim-classifier/v15/model.tar.gz
  claim-classifier/v16/model.tar.gz
  claim-classifier/v17/model.tar.gz   <- currently deployed
  isi-classifier/v8/model.tar.gz
  proof-reader/v4/model.tar.gz
```

S3 is also where training data snapshots live, so a given model artifact's `vN` path and its
corresponding training-data snapshot path together give you full lineage: this exact model came from
this exact data with this exact code version.

## Single vs. multi-model endpoints

A **single-model endpoint** hosts exactly one model per deployed endpoint — simple, but means every
model (claim classifier, ISI classifier, proof-reader) needs its own always-on compute instance,
which gets expensive and wasteful if any individual model's traffic is low or spiky.

A **multi-model endpoint (MME)** hosts many models behind one endpoint, on shared compute, loading
models into memory on demand and evicting less-recently-used ones when memory pressure requires it —
conceptually similar to an LRU cache, but for model weights instead of data. The request specifies
which model to invoke (via a target-model parameter), and Sagemaker's MME container handles
loading/caching transparently.

```python
predictor.predict(
    data=payload,
    target_model="claim-classifier/v17/model.tar.gz",  # which model to route this request to
)
```

**Why this matters for this project specifically:** the four modules (Claim Extraction &
Classification, Proof Reading, ISI Classification, Content Comparator) are architecturally distinct
models with very different traffic patterns — Claim Classification likely sees far more traffic than
ISI Classification, since every extracted claim gets classified but ISI checking happens once per
document. Running four separate always-on single-model endpoints means paying for four sets of idle
capacity sized for peak load; a multi-model endpoint shares compute across all four and only pays for
the peak of the *combined* traffic, at the cost of a small latency hit on a cold model load (the
first request for a model that isn't currently in memory has to wait for it to load) and less
per-model isolation (a memory-hungry model can evict others more aggressively).

## Auto-scaling for load balancing

"Implemented Auto-scaling to facilitate load balancing for single/multi model endpoints" refers to
**Sagemaker endpoint auto-scaling**: attaching a scaling policy to an endpoint's instance count (or,
for serverless inference, its concurrency) that reacts to a live metric — most commonly
`InvocationsPerInstance` or CPU/GPU utilization from CloudWatch.

```python
import boto3

client = boto3.client("application-autoscaling")

client.register_scalable_target(
    ServiceNamespace="sagemaker",
    ResourceId="endpoint/claim-classifier-endpoint/variant/AllTraffic",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    MinCapacity=1,
    MaxCapacity=6,
)

client.put_scaling_policy(
    PolicyName="claim-classifier-target-tracking",
    ServiceNamespace="sagemaker",
    ResourceId="endpoint/claim-classifier-endpoint/variant/AllTraffic",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    PolicyType="TargetTrackingScaling",
    TargetTrackingScalingPolicyConfiguration={
        "TargetValue": 70.0,   # target ~70 invocations/instance/min
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance"
        },
    },
)
```

Target-tracking scaling is the standard pattern: you pick a target value for a metric, and AWS adds
or removes instances to keep the metric near that target — scale out under a content-review deadline
crunch (a big content batch dropped for review), scale back in overnight when traffic is low. This is
the load-balancing story: incoming requests get distributed across however many instances are
currently running behind the endpoint, and the instance count itself flexes with demand instead of
being sized (and paid for) at fixed peak capacity.

## Multi-client production considerations: Eli Lilly vs. AstraZeneca

This platform runs in production for two pharma clients, Eli Lilly and AstraZeneca, and the
multi-model endpoint (MME) picture above gets a real constraint once you add a second client: an MME
is a great way to share compute *across a single client's own models* (claim classifier, ISI
classifier, proof-reader), but it is the wrong place to also mix *across clients*. Two things follow
from that.

**Namespacing model artifacts and endpoints per client.** If Eli Lilly's and AstraZeneca's fine-tuned
claims classifiers diverge — different training data, different claim taxonomies, or even just
different fine-tuning schedules — they need to be addressed as distinct artifacts, not variants of the
same one:

```
s3://claims-eli-lilly/model-artifacts/claim-classifier/v17/model.tar.gz
s3://claims-astrazeneca/model-artifacts/claim-classifier/v9/model.tar.gz
```

and deployed behind **separate multi-model endpoints per client** (`lilly-claims-mme`,
`astrazeneca-claims-mme`) rather than one shared MME loading both clients' models by
`target_model` name. This costs some of the cross-model compute-sharing efficiency described above,
but it's the right trade: an MME evicts less-recently-used models under memory pressure, and you do
not want AstraZeneca's traffic volume determining whether Eli Lilly's model stays warm in memory (or
vice versa) — that's a cross-client noisy-neighbor problem, not just a performance nuisance, given
each client's SLA is contractually independent of the other's. Separate endpoints also make the IAM
boundary clean: each endpoint's invoke permission is scoped to that client's Lambda routing role, so
there's no code path where a misconfigured request could invoke the wrong client's model.

**Auto-scaling policy per client, sized to that client's own traffic pattern.** Eli Lilly and
AstraZeneca submit content for review on their own independent schedules — a large campaign launch or
an end-of-quarter content push for one client is not correlated with the other's submission volume.
If both clients shared a single auto-scaled endpoint, a submission spike from one client would consume
the scaled-up capacity and could starve the other client's requests of the low-latency response their
SLA assumes, until the target-tracking policy catches up. Giving each client its own scalable target
(own `MinCapacity`/`MaxCapacity`, own target-tracking policy tuned to that client's typical and peak
`InvocationsPerInstance`) means one client's load spike scales that client's endpoint independently,
without touching the other's capacity or cost:

```python
# One scalable target and policy per client endpoint — not shared
for client, endpoint in [("lilly", "lilly-claims-mme"), ("astrazeneca", "astrazeneca-claims-mme")]:
    client_autoscale.register_scalable_target(
        ServiceNamespace="sagemaker",
        ResourceId=f"endpoint/{endpoint}/variant/AllTraffic",
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MinCapacity=1,
        MaxCapacity=6,   # tuned independently per client's observed peak
    )
```

The general principle carries beyond this project: whenever "multi-tenant" and "shared infrastructure
for cost efficiency" show up together, the auto-scaling and isolation design has to make sure sharing
compute never turns into one tenant borrowing another tenant's SLA headroom.

## EventBridge scheduled health checks

**EventBridge Scheduler** fires events on a schedule (cron-like or rate-based) — here, used to
periodically invoke a lightweight health-check request against each live endpoint and confirm it's
responding correctly (not just "is the instance up" but "does invoking it with a known test input
return the expected shape of output"). This is a cheap, proactive way to catch an unhealthy or
misconfigured endpoint before real user traffic hits it, and pairs naturally with CloudWatch alarms —
a failed scheduled health check can itself publish a custom metric that triggers an alert or even a
rollback to the previous model version.

```python
# Simplified shape of what the scheduled health-check Lambda does
def health_check_handler(event, context):
    response = sagemaker_runtime.invoke_endpoint(
        EndpointName="claim-classifier-endpoint",
        Body=json.dumps({"text": "Drug X reduces symptom severity by 30% vs placebo."}),
        ContentType="application/json",
    )
    result = json.loads(response["Body"].read())
    assert "labels" in result and "scores" in result   # expected output shape
    cloudwatch.put_metric_data(
        Namespace="ClaimsPipeline",
        MetricData=[{"MetricName": "HealthCheckSuccess", "Value": 1, "Unit": "Count"}],
    )
```

## Lambda + API Gateway as the deployment/routing layer

Consumers of these models (an internal content-review web app, say) don't call the Sagemaker endpoint
directly. **API Gateway** exposes a stable REST endpoint to clients; **Lambda** sits behind it as the
integration layer that does request validation/transformation, calls `invoke_endpoint` against the
right Sagemaker endpoint (routing to the right `target_model` for a multi-model endpoint), and shapes
the response. This layering buys you: authentication/authorization at the API Gateway layer, the
ability to change or version the underlying Sagemaker endpoint without changing the client-facing
contract, and a natural place to plug in request logging, rate limiting, and light business logic
(e.g., only forward to the model if the document passed an earlier validation stage) without touching
the model-serving infrastructure itself.

## Two supporting services worth naming

- **AWS Secrets Manager**: stores and rotates credentials the Lambda layer needs (e.g., internal
  service auth tokens, third-party API keys) rather than hardcoding them — Lambda fetches secrets at
  invocation time rather than having them baked into deployed code.
- **AWS CloudWatch**: the observability backbone underneath everything above — endpoint invocation
  counts, latency, error rates, and the custom health-check metric all flow into CloudWatch, which is
  what the auto-scaling policy and any alerting ultimately reads from. If asked "how would you know
  this system is unhealthy," CloudWatch dashboards/alarms are the answer.
- **AWS Comprehend** is worth mentioning as a related-but-different tool: it's AWS's managed NLP
  service (entity recognition, sentiment, key-phrase extraction) usable out of the box without
  training your own model. It's a reasonable option for lighter-weight text tasks, but claim
  classification and ISI classification need custom label taxonomies specific to pharma content, which
  is exactly the case where a fine-tuned custom model (chapter 02) earns its keep over an off-the-shelf
  managed API.

## Tying it back

Put together, this is the full deployment loop from the chapter 00 MLOps diagram: a Sagemaker
training job (fine-tuning) writes a versioned artifact to S3 → that artifact gets deployed to a
single- or multi-model Sagemaker endpoint with an auto-scaling policy attached → EventBridge fires
scheduled health checks against the live endpoint → Lambda + API Gateway is the client-facing routing
layer that actually serves predictions → CloudWatch and Secrets Manager are the observability and
credential-management backbone running underneath all of it. Notebook
`04_sagemaker_pipeline_mock_demo.ipynb` implements a fully offline mock of the estimator/deploy/route
API shape so you can see the mental model run end-to-end without needing real AWS credentials.

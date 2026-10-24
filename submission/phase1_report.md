# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
50
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->244.2<!--/answer--> | <!--answer:vm_warm_median_ms-->211.0<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->1413.4<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->635.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1288.0<!--/answer--> | <!--answer:function_warm_median_ms-->635.1<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm       cold=  244.2ms  warm_median=  211.0ms
#############
[OK  ] cloudrun  cold= 1413.4ms  warm_median=  635.8ms
########################################
[OK  ] function  cold= 1288.0ms  warm_median=  635.1ms
########################################
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
- **IaaS VM:** Nothing scales automatically. It runs on a single fixed instance (`--machine-type=e2-micro`). Concurrent requests compete for the same CPU and memory resources on that instance, leading to latency degradation or timeouts if capacity is exceeded.
- **Cloud Run:** Automatically scales instances out based on incoming concurrent request traffic. Multiple requests can be processed concurrently per container instance, and new container instances are dynamically spun up up to the hard limit of `--max-instances=3`.
- **Cloud Function:** Scales automatically based on incoming HTTP event triggers, dynamically provisioning execution environments to absorb traffic spikes and scaling back down to zero when idle.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
- **IaaS VM:** High effort. Required manually creating a custom VPC, subnet, firewall rules for SSH and port 8080, setting up a VM instance, writing a startup script to install Docker, configuring authentication for Artifact Registry, and managing container lifecycles manually.
- **Cloud Run:** Low effort. Built and pushed a container image to Artifact Registry, then deployed it using a single managed CLI command (`gcloud run deploy`), abstracting infrastructure management, TLS, and autoscaling.
- **Cloud Function:** Lowest effort. Deployed directly from source code using buildpacks, removing the need to write a Dockerfile or manage image builds while handling serverless triggers out-of-the-box.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
SERVICE   REGION      URL                                            LAST DEPLOYED BY   LAST DEPLOYED AT
echo      us-central1 https://echo-jyo4maq6aq-uc.a.run.app           ...
echo-bot  us-central1 https://echo-bot-1088631253388.us-central1.run.app ...
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output of `gcloud run services list` shows that Google Cloud Functions (Gen 2) executes directly on underlying Cloud Run services. When deploying from source code, Google Cloud packages it and automatically provisions a corresponding Cloud Run service. This shared architectural foundation explains why the cold-start latencies (~1.3s–1.4s) and warm latencies (~635ms) for both platforms are nearly identical. Both run within the same container runtime environment and follow identical scale-to-zero lifecycles. The core difference lies in the packaging workflow: Cloud Run requires a pre-built container image, whereas Cloud Functions accepts raw source code and automates containerization via buildpacks.
<!--/answer-->

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
48
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
| IaaS VM | <!--answer:vm_cold_ms-->277.3<!--/answer--> | <!--answer:vm_warm_median_ms-->211.3<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->2232.2<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->631.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->1357.3<!--/answer--> | <!--answer:function_warm_median_ms-->633.3<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  277.3ms  warm_median=  211.3ms
            #############
[OK  ] cloudrun  cold= 2232.2ms  warm_median=  631.8ms
            ########################################
[OK  ] function  cold= 1357.3ms  warm_median=  633.3ms
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
For the IaaS VM, nothing is created automatically to handle scaling, remaining a single instance, unless a separate scaling mechanism is configured. Its `--machine-type=e2-micro` setting limits the CPU and memory available to that instance, constraining its ability to handle concurrent requests. 
Cloud Run automatically creates additional container instances in response to demand, up to the limit configured by `--max-instances=3`. 
Cloud Functions (Gen 2) also scales automatically, creating additional function instances as demand increases. Its effective scaling limit depends on the function's instance configuration and applicable quotas.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
For the IaaS VM, deployment required provisioning the VM, configuring the custom VPC and subnet, setting firewall rules, installing Docker, and pulling and running the container image. This provided direct control over the machine and network configuration but also introduced operational responsibilities. 
For Cloud Run, a pre-built container image from Artifact Registry was used, while Google managed the serving infrastructure, HTTPS endpoint and automatic scaling. 
For Cloud Functions, the source code (`main.py` and `requirements.txt`) was deployed, while the platform handled the build, container packaging and deployment. This reduced the need to manage container infrastructure directly, although appropriate build permissions and runtime configuration were still required.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
✔
SERVICE: echo
REGION: us-central1
URL: https://echo-676372265360.us-central1.run.app
LAST DEPLOYED BY: service-676372265360@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-25T09:59:35.337650Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-676372265360.us-central1.run.app
LAST DEPLOYED BY: luca.corrado@eurecom.fr
LAST DEPLOYED AT: 2026-10-25T09:51:24.169123Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The `gcloud run services list` output shows two distinct Cloud Run services: `echo`, deployed by the Google Cloud Functions service agent, and `echo-bot`, deployed by the user account.  This is consistent with `echo` being the Gen-2 Cloud Function, which Google builds into a container and runs as a managed Cloud Run service. The two deployments, thus, share the underlying serving platform, helping explain their nearly identical warm median latencies (633.3 ms for the function versus 631.8 ms for Cloud Run). However, their measured cold-start latencies differ substantially (1357.3 ms versus 2232.2 ms), so shared infrastructure does not imply identical startup behaviour. The main distinction is the deployment abstraction: Cloud Run accepts a container image directly, whereas the function deployment starts from source code and automates the build and packaging process. The service listing supports this interpretation, although function metadata would provide stronger confirmation.
<!--/answer-->

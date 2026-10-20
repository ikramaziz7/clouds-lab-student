# Phase 1 — Comparison Report

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
49
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

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->275.3<!--/answer--> | <!--answer:vm_warm_median_ms-->210.9<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->699.1<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->641.8<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->748.1<!--/answer--> | <!--answer:function_warm_median_ms-->670.8<!--/answer--> |

<!--answer:latency_plot-->
[vm      ]  cold=  275.3ms  warm_median=  210.9ms  #############
[cloudrun]  cold=  699.1ms  warm_median=  641.8ms  ######################################
[function]  cold=  748.1ms  warm_median=  670.8ms  ########################################
<!--/answer-->

---

## 3. Per-platform analysis

<!--answer:scaling_behaviour-->
The VM does not scale at all. One container runs on one machine, and all requests are
handled by that single instance. If traffic spikes, requests simply queue up. There is
no automatic scaling unless you set it up yourself, which goes against the point of IaaS.

Cloud Run scales automatically. When traffic increases, it spins up new container
instances to handle the load. In this deployment, the maximum was capped at 3 instances
(--max-instances=3). When there is no traffic, it scales down to zero, which is why cold
starts happen.

Cloud Functions also scales automatically, one instance per request. Since we did not
set a max-instances limit, GCP manages scaling on its own. Like Cloud Run, it scales to
zero when idle, so cold starts are expected after a period of no traffic.
<!--/answer-->

<!--answer:deployment_effort-->
The VM required the most work by far. We had to build and push a Docker image, create
a custom VPC with subnet and firewall rules, provision a VM, SSH into it, install Docker,
authenticate to Artifact Registry, and manually run the container. Any future update
would require SSH-ing back in and restarting the container by hand. It gives full
control, but at the cost of a lot of manual steps.

Cloud Run was much easier. After building the Docker image, a single gcloud run deploy
command handled everything else — TLS, networking, scaling, and traffic routing. No SSH
required, no firewall rules to manage.

Cloud Functions was the easiest. We just pointed gcloud at the source code and it
handled everything, including containerisation. The trade-off is less flexibility over
the runtime environment, but for a simple app like this echo bot, it is more than enough.
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-1046881594014.us-central1.run.app
LAST DEPLOYED BY: service-1046881594014@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-20T13:47:12.812047Z

SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-1046881594014.us-central1.run.app
LAST DEPLOYED BY: ikram.aziz@eurecom.fr
LAST DEPLOYED AT: 2026-10-20T13:42:18.811547Z
<!--/answer-->

<!--answer:convergence_explanation-->
The output shows two Cloud Run services: echo-bot (which we deployed directly) and echo
(deployed by gcf-admin-robot, the Cloud Functions service agent). This reveals something
interesting: Cloud Functions gen 2 is not a separate runtime — it is actually built on
top of Cloud Run. When you deploy a function, GCP automatically creates and manages a
Cloud Run service behind the scenes.

This explains why the cold-start times for Cloud Run (699ms) and Cloud Function (748ms)
are almost identical. Both ultimately run as Cloud Run containers that scale to zero,
so they pay the same container startup cost. The small difference (~49ms) comes down to
network variance and the thin extra layer of the function framework on top.

The real difference between the two is not the runtime, but the abstraction. Cloud Run
requires you to build and push a Docker image yourself. Cloud Functions takes your source
code and handles the containerisation for you. Under the hood though, they converge to
the same thing.
<!--/answer-->

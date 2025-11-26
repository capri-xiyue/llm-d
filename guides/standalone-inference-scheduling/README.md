# Well-lit Path: Intelligent Standalone Inference Scheduling

## Overview

This guide deploys the recommended out of the box [scheduling configuration](https://github.com/llm-d/llm-d-inference-scheduler/blob/main/docs/architecture.md) for most vLLM deployments, reducing tail latency and increasing throughput through load-aware and prefix-cache aware balancing. This can be run on a single GPU that can load [Qwen/Qwen3-0.6B](https://huggingface.co/Qwen/Qwen3-0.6B).

This profile defaults to the approximate prefix cache aware scorer, which only observes request traffic to predict prefix cache locality. The [precise prefix cache aware routing feature](../precise-prefix-cache-aware) improves hit rate by introspecting the vLLM instances for cache entries and will become the default in a future release.

## Hardware Requirements

This example out of the box requires 2 GPUs of any supported kind:
- **NVIDIA GPUs**: Any NVIDIA GPU (support determined by the inferencing image used)
- **Intel XPU/GPUs**: Intel Data Center GPU Max 1550 or compatible Intel XPU device
- **TPUs**: Google Cloud TPUs (when using GKE TPU configuration)

**Alternative CPU Deployment**: For CPU-only deployment (no GPUs required), see the [Hardware Backends](#hardware-backends) section for CPU-specific deployment instructions. CPU deployment requires Intel/AMD CPUs with 64 cores and 64GB RAM per replica.

## Prerequisites

- Have the [proper client tools installed on your local system](../prereq/client-setup/README.md) to use this guide.
- Ensure your cluster infrastructure is sufficient to [deploy high scale inference](../prereq/infrastructure)
- [Create the `llm-d-hf-token` secret in your target namespace with the key `HF_TOKEN` matching a valid HuggingFace token](../prereq/client-setup/README.md#huggingface-token) to pull models.
- Have the [Monitoring stack](../../docs/monitoring/README.md) installed on your system.


## Installation

Use the helmfile to compose and install the stack. The Namespace in which the stack will be deployed will be derived from the `${NAMESPACE}` environment variable. If you have not set this, it will default to `llm-d-inference-scheduler` in this example.

**_IMPORTANT:_** When using long namespace names (like `llm-d-inference-scheduler`), the generated pod hostnames may become too long and cause issues due to Linux hostname length limitations (typically 64 characters maximum). It's recommended to use shorter namespace names (like `llm-d`) and set `RELEASE_NAME_POSTFIX` to generate shorter hostnames and avoid potential networking or vLLM startup problems.

**For GPU deployment (default):**
```bash
export NAMESPACE=llm-d-standalone-inference-scheduler # or any other namespace (shorter names recommended)
kubectl create namespace ${NAMESPACE}

# Clone the repo and switch to the latest release tag 
tag=$(curl -s https://api.github.com/repos/llm-d/llm-d/releases/latest | jq -r '.tag_name')
git clone https://github.com/llm-d/llm-d.git && cd llm-d && git checkout "$tag"

cd guides/standalone-inference-scheduling
helmfile apply -n ${NAMESPACE}
```

**_NOTE:_** You can set the `$RELEASE_NAME_POSTFIX` env variable to change the release names. This is how we support concurrent installs. Ex: `RELEASE_NAME_POSTFIX=standalone-inference-scheduling-2 helmfile apply -n ${NAMESPACE}`



## Verify the Installation

- Firstly, you should be able to list all helm releases to view the 3 charts got installed into your chosen namespace:

```bash
helm list -n ${NAMESPACE}
NAME                        NAMESPACE                 REVISION  UPDATED                               STATUS    CHART                     APP VERSION
gaie-standalone-inference-scheduling   llm-d-standalone-inference-scheduler 1         2025-08-24 11:24:53.231918 -0700 PDT  deployed  inferencepool-v1.2.0-rc.1 v1.2.0-rc.2
ms-standalone-inference-scheduling     llm-d-inference-scheduler 1         2025-08-24 11:24:58.360173 -0700 PDT  deployed  llm-d-modelservice-v0.3.8 v0.3.0
```

- Out of the box with this example you should have the following resources:

```bash
kubectl get all -n ${NAMESPACE}
NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/gaie-standalone-inference-scheduling-epp-f8fbd9897-cxfvn                     1/1     Running   0          3m59s
pod/ms-standalone-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b58lw9   2/2     Running   0          3m55s
pod/ms-standalone-inference-scheduling-llm-d-modelservice-decode-8ff7fd5bt5f9s   2/2     Running   0          3m55s

NAME                                                         TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
service/gaie-standalone-inference-scheduling-epp                        ClusterIP      10.16.3.151   <none>        9002/TCP,9090/TCP              3m59s
service/gaie-standalone-inference-scheduling-ip-18c12339                ClusterIP      None          <none>        54321/TCP                      3m59s

NAME                                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/gaie-standalone-inference-scheduling-epp                        1/1     1            1           4m
deployment.apps/ms-standalone-inference-scheduling-llm-d-modelservice-decode    2/2     2            2           3m56s

NAME                                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/gaie-standalone-inference-scheduling-epp-f8fbd9897                        1         1         1       4m
replicaset.apps/ms-standalone-inference-scheduling-llm-d-modelservice-decode-8ff7fd5b8    2         2         2       3m56s
```

**_NOTE:_** This assumes no other guide deployments in your given `${NAMESPACE}` and you have not changed the default release names via the `${RELEASE_NAME}` environment variable.

## Using the stack

For instructions on getting started making inference requests see [our docs](../../docs/getting-started-inferencing.md)

## Cleanup

To remove the deployment:

```bash
# From examples/standalone-inference-scheduling
helmfile destroy -n ${NAMESPACE}

# Or uninstall manually
helm uninstall gaie-standalone-inference-scheduling -n ${NAMESPACE}
helm uninstall ms-syandalone-inference-scheduling -n ${NAMESPACE}
```

**_NOTE:_** If you set the `$RELEASE_NAME_POSTFIX` environment variable, your release names will be different from the command above: `infra-$RELEASE_NAME_POSTFIX`, `gaie-$RELEASE_NAME_POSTFIX` and `ms-$RELEASE_NAME_POSTFIX`.

## Customization

For information on customizing a guide and tips to build your own, see [our docs](../../docs/customizing-a-guide.md)

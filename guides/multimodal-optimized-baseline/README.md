# Multimodal Optimized Baseline Guide

This guide deploys the recommended [configuration](https://github.com/llm-d/llm-d-router/blob/main/docs/architecture.md) for multimodal vLLM deployments, reducing tail latency and increasing throughput through load-aware and prefix-cache aware balancing.

The multimodal-optimized-baseline defaults to two main routing criteria:
* **Prefix-cache aware:** Scores candidate endpoints by estimating multimodal prompt prefix cache reuse (e.g., matching text + image hashes) on each model server.
* **Load-aware:** Scores candidate endpoints based on queue depth and kv-cache utilization to prevent server bottlenecks.

---

## Default Configuration

| Parameter          | Value                                                   |
| ------------------ | ------------------------------------------------------- |
| Default Model      | [Qwen/Qwen3-32B](https://huggingface.co/Qwen/Qwen3-32B) |
| Replicas           | 8                                                       |
| Tensor Parallelism | 2                                                       |
| GPUs per replica   | 2                                                       |
| Total GPUs         | 16                                                      |

### Supported Hardware Backends

This guide includes configurations for the following accelerators and inference backends:

| Backend             | Directory Overlay | Notes |
| ------------------- | ----------------- | ----- |
| AMD GPU (vLLM)      | `modelserver/amd/vllm/` | ROCm-compatible AMD GPUs running vLLM |
| Intel XPU           | `modelserver/xpu/vllm/` | Intel Data Center GPU Max 1550+ |
| Intel Gaudi (HPU)   | `modelserver/hpu/vllm/` | Gaudi 1/2/3 with resource claim support |
| Google TPU v6e      | `modelserver/tpu-v6/vllm/` | GKE Google TPU v6e running vLLM |
| Google TPU v7       | `modelserver/tpu-v7/vllm/` | GKE Google TPU v7 running vLLM |
| CPU                 | `modelserver/cpu/vllm/` | x86 CPUs running vLLM (development / test setups) |

---

## Prerequisites

1. Install the local client tooling using the [client setup guide](../../helpers/client-setup/README.md).
2. Clone and check out the llm-d repository:
   ```bash
   export branch="main" # branch, tag, or commit hash
   git clone https://github.com/llm-d/llm-d.git && cd llm-d && git checkout ${branch}
   ```
3. Set up environment variables:
   ```bash
   export GAIE_VERSION=v1.5.0
   export ROUTER_CHART_VERSION=v0
   export GUIDE_NAME="multimodal-optimized-baseline"
   export NAMESPACE=llm-d-multimodal-optimized-baseline
   ```
4. Install the Gateway API Inference Extension CRDs:
   ```bash
   kubectl apply -k "https://github.com/kubernetes-sigs/gateway-api-inference-extension/config/crd?ref=${GAIE_VERSION}"
   ```
5. Create the namespace:
   ```bash
   kubectl create namespace ${NAMESPACE}
   ```

---

## Installation Instructions

### 1. Deploy the llm-d Router

#### Standalone Mode
Deploy the llm-d Router in **Standalone Mode** overlaying router custom configurations:
```bash
# Run from the root of the llm-d repo
helm install ${GUIDE_NAME} \
    oci://ghcr.io/llm-d/charts/llm-d-router-standalone-dev \
    -f guides/recipes/router/base.values.yaml \
    -f guides/${GUIDE_NAME}/router/${GUIDE_NAME}.values.yaml \
    -n ${NAMESPACE} --version ${ROUTER_CHART_VERSION}
```

<details>
<summary><h4>Gateway Mode</h4></summary>

To use a Kubernetes Gateway managed proxy rather than the standalone version, follow these steps:

1. _Deploy a Kubernetes Gateway_ named by following one of [the gateway guides](../prereq/gateways).
2. _Deploy the llm-d router and an HTTPRoute_ that connects it to the Gateway as follows:

```bash
export PROVIDER_NAME=gke # options: none, gke, agentgateway, istio
helm install ${GUIDE_NAME} \
    oci://ghcr.io/llm-d/charts/llm-d-router-gateway-dev  \
    -f guides/recipes/router/base.values.yaml \
    -f guides/${GUIDE_NAME}/router/${GUIDE_NAME}.values.yaml \
    --set provider.name=${PROVIDER_NAME} \
    --set httpRoute.create=true \
    --set httpRoute.inferenceGatewayName=llm-d-inference-gateway \
    -n ${NAMESPACE} --version ${ROUTER_CHART_VERSION}
```

</details>

### 2. Deploy the Model Server

Apply the Kustomize overlays matching your chosen backend tier. For example, to deploy on AMD GPU using vLLM:
```bash
kubectl apply -n ${NAMESPACE} -k guides/${GUIDE_NAME}/modelserver/amd/vllm/
```

To deploy on Intel Gaudi HPUs:
```bash
kubectl apply -n ${NAMESPACE} -k guides/${GUIDE_NAME}/modelserver/hpu/vllm/
```

---

## Verification

### 1. Retrieve the Proxy Endpoint IP

**Standalone Mode:**
```bash
export IP=$(kubectl get service ${GUIDE_NAME}-epp -n ${NAMESPACE} -o jsonpath='{.spec.clusterIP}')
```

<details>
<summary><b>Gateway Mode:</b></summary>

```bash
export IP=$(kubectl get gateway llm-d-inference-gateway -n ${NAMESPACE} -o jsonpath='{.status.addresses[0].value}')
```

</details>

### 2. Send a Multimodal Test Request

Open a debug container within the cluster namespace:
```bash
kubectl run curl-debug --rm -it \
    --image=cfmanteiga/alpine-bash-curl-jq \
    --env="IP=$IP" \
    --env="NAMESPACE=$NAMESPACE" \
    -- /bin/bash
```

Send an OpenAI-compatible Chat Completion request containing a text prompt and a target image URL:
```bash
curl -X POST http://${IP}/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{
        "model": "Qwen/Qwen3-32B",
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": "What details are present in this photo?"
                    },
                    {
                        "type": "image_url",
                        "image_url": {
                            "url": "https://picsum.photos/800/600"
                        }
                    }
                ]
            }
        ]
    }' | jq
```

---

## Cleanup

To tear down and clean up all deployed resources:
```bash
helm uninstall ${GUIDE_NAME} -n ${NAMESPACE}
kubectl delete -n ${NAMESPACE} -k guides/${GUIDE_NAME}/modelserver/amd/vllm/
kubectl delete namespace ${NAMESPACE}
```

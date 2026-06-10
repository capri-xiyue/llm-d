Multimodal Serving

The **multimodal-serving well-lit path** is a horizontal, workload-centric umbrella that serves
multimodal *programs* on llm-d by composing the capability paths into a stack and exposing a ladder
of deployment options that trade complexity for capability. For the workload model, canonical
shapes, and the direction this path is driving toward, see the
[Multimodal Serving well-lit path](../../docs/well-lit-paths/multimodal-serving.md); this guide is
the operational counterpart.


## The Optimization Stack

The guide composes llm-d's capability paths into layers that each relieve a specific pressure of
the multimodal workload; the deployment options below compose them.

| Layer | What it does for the workload |
| :--- | :--- |
| **[Optimized baseline](../optimized-baseline/README.md)** — routing foundation | Prefix-cache scorer routes a turn to the replica already holding its prefix; load-aware scorers keep bursts off hot replicas. Every option starts here. |


## Deployment Options

The layers above compose into deployment options spanning a range of capability and operational
cost — from a routing-and-offloading baseline up to e-disaggregated serving — added incrementally
as a workload's scale and latency targets grow. Concrete, benchmarked options are landing in
this directory as sub-guides.

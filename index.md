---
layout: default
---

# RHOAI Models-as-a-Service

Deploy API-key-governed LLM access on OpenShift AI 3.4 with Kuadrant auth, rate-limiting, and a shared Gateway.

## Documentation

- [Architecture guide — how MaaS components connect](docs/how-maas-works.html)
- [Deploy steps (README on GitHub)](https://github.com/nirjhar17/maas_latest#steps)
- [RHOAI 3.4 MaaS official docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/index)
- [Companion lab guide](https://rh-aiservices-bu.github.io/rhoai-maas-guide/modules/main/index.html)
- [James Harmison from-scratch install](https://github.com/jharmison-redhat/maas-from-scratch) (optional extras: custom CA, CNPG TLS)

## Quick start

Prerequisites: OpenShift 4.19+ with `cluster-admin`, `oc`, `envsubst`, `jq`, and `curl` on PATH.

1. Install operators: `oc apply -k manifests/01-prerequisites/operators/`
2. Configure platform: `oc apply -k manifests/02-platform-config/`
3. Deploy MaaS platform: `oc apply -k manifests/03-maas-platform/`
4. Deploy models: `oc apply -k manifests/05-maas-models/simulator/` (or another model bundle)
5. Verify: `manifests/06-verification/verify.sh`

See the [full README](https://github.com/nirjhar17/maas_latest) for MetalLB, GPU setup, and model-specific steps.

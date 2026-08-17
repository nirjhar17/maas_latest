# RHOAI Models-as-a-Service (MaaS)

Deploy API-key-governed LLM access on OpenShift AI 3.4 with Kuadrant auth/rate-limiting and a shared Gateway.

Documentation:
1. [RHOAI 3.4 MaaS Official Docs](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/govern_llm_access_with_models-as-a-service/index)
2. [Companion lab guide](https://rh-aiservices-bu.github.io/rhoai-maas-guide/modules/main/index.html)
3. [Architecture & how components connect](docs/how-maas-works.html)
4. [James Harmison from-scratch install](https://github.com/jharmison-redhat/maas-from-scratch) (optional extras: custom CA, CNPG TLS)

---

### Prerequisites

- OpenShift 4.19+ cluster with `cluster-admin`
- `oc`, `envsubst`, `jq`, `curl` on PATH
- Registry pull secret for `registry.redhat.io` (GPU models need it; simulator does not)

---

### Steps

**1. Install required operators**

```
oc apply -k manifests/01-prerequisites/operators/
```

Wait for all four CSVs to succeed before continuing:

```
oc wait csv -n redhat-ods-operator \
  -l operators.coreos.com/rhods-operator.redhat-ods-operator="" \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s

oc wait csv -n openshift-operators \
  -l operators.coreos.com/rhcl-operator.openshift-operators="" \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s

oc wait csv -n cert-manager-operator \
  -l operators.coreos.com/openshift-cert-manager-operator.cert-manager-operator="" \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s

oc wait csv -n openshift-lws-operator \
  -l operators.coreos.com/leader-worker-set.openshift-lws-operator="" \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s
```

Note: On AWS/Azure/GCP skip the MetalLB step below. On baremetal, OpenStack, or SNO you need it.

**1a. (Non-cloud only) Install MetalLB**

```
oc apply -k manifests/01-prerequisites/metallb/
oc wait csv -n metallb-system \
  -l operators.coreos.com/metallb-operator.metallb-system="" \
  --for=jsonpath='{.status.phase}'=Succeeded --timeout=120s
oc apply -f manifests/01-prerequisites/metallb/metallb.yaml
```

Compute an unused IP one above the first node's internal IP and create the pool:

```
NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
METALLB_IP=$(echo "$NODE_IP" | awk -F. '{printf "%s.%s.%s.%d",$1,$2,$3,$4+1}')
export METALLB_IP_RANGE="${METALLB_IP}-${METALLB_IP}"
envsubst < manifests/03-maas-platform/openshift-gateway-setup/metallb-config.yaml | oc apply -f -
```

**1b. (Optional) Install GPU operators**

Required only if you plan to run granite-tiny-gpu, gemma, or gpt-oss-20b.

```
oc apply -k manifests/01-prerequisites/gpu/
# Wait for NFD and NVIDIA CSVs, then create operand instances:
oc apply -k manifests/01-prerequisites/gpu/instances/
oc wait clusterpolicy gpu-cluster-policy \
  --for=jsonpath='{.status.state}'=ready --timeout=600s
```

---

**2. Configure Kuadrant, Authorino, and User Workload Monitoring**

Apply the three Kuadrant pieces separately — do NOT use `oc apply -k` on the whole directory in one shot. The Kuadrant operator auto-creates the Authorino CR, so TLS must be patched in afterwards.

```
oc apply -f manifests/02-platform-config/kuadrant/namespace.yaml
oc apply -f manifests/02-platform-config/kuadrant/service-annotation.yaml
oc apply -f manifests/02-platform-config/kuadrant/kuadrant.yaml
oc wait --for=condition=Ready kuadrant/kuadrant -n kuadrant-system --timeout=120s
```

Patch Authorino to enable TLS using the service-ca-generated secret:

```
oc patch authorino authorino -n kuadrant-system --type=merge --patch '{
  "spec": {"listener": {"tls": {"enabled": true,
    "certSecretRef": {"name": "authorino-server-cert"}}}}}'

oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt

oc wait --for=condition=Available deployment/authorino \
  -n kuadrant-system --timeout=300s
```

Enable User Workload Monitoring:

```
oc apply -k manifests/02-platform-config/uwm/
oc wait --for=condition=Available deployment/prometheus-operator \
  -n openshift-user-workload-monitoring --timeout=300s
```

---

**3. Create GatewayClass and MaaS Gateway**

```
oc apply -f manifests/02-platform-config/gatewayclass.yaml
oc wait --for=condition=Accepted gatewayclass/openshift-default --timeout=120s
```

Apply the `2Gi` memory ConfigMap (prevents OOMKill when Wasm extensions load):

```
oc apply -f manifests/02-platform-config/gateway-resources.yaml
```

Render and apply the Gateway (uses `envsubst` for cluster-specific values):

```
export CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster \
  -o jsonpath='{.spec.domain}')
export CERT_NAME=$(oc get ingresscontroller default \
  -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate.name}' 2>/dev/null)
export CERT_NAME="${CERT_NAME:-router-certs-default}"

envsubst '${CLUSTER_DOMAIN} ${CERT_NAME}' \
  < manifests/02-platform-config/gateway.yaml.tmpl | oc apply -f -

oc wait --for=condition=Programmed gateway/maas-default-gateway \
  -n openshift-ingress --timeout=120s
```

Label the applications namespace for gateway access:

```
oc label namespace redhat-ods-applications \
  maas.opendatahub.io/gateway-access=true --overwrite
```

**3a. (Non-cloud only) Create passthrough Route**

```
envsubst '${CLUSTER_DOMAIN}' \
  < manifests/03-maas-platform/openshift-gateway-setup/route.yaml.tmpl | oc apply -f -
```

---

**4. Deploy PostgreSQL**

The MaaS API needs a database before the DSC enables `modelsAsService`. Create the secrets imperatively (never commit credentials):

```
POSTGRES_PASSWORD=$(openssl rand -base64 16 | tr -d '=+/')

oc create secret generic postgres-creds \
  -n redhat-ods-applications \
  --from-literal=POSTGRES_USER=maas \
  --from-literal=POSTGRES_PASSWORD="${POSTGRES_PASSWORD}" \
  --from-literal=POSTGRES_DB=maas

oc create secret generic maas-db-config \
  -n redhat-ods-applications \
  --from-literal=DB_CONNECTION_URL="postgresql://maas:${POSTGRES_PASSWORD}@postgres:5432/maas?sslmode=disable"

oc apply -k manifests/03-maas-platform/
oc wait --for=condition=Available deployment/postgres \
  -n redhat-ods-applications --timeout=120s
```

Note: Two separate secrets — `postgres-creds` is consumed by the PostgreSQL pod; `maas-db-config` is consumed by maas-api. The connection URL uses `sslmode=disable` (plain Deployment, not CNPG).

---

**5. Configure RHOAI (DSCI + DSC)**

Apply in two stages because `OdhDashboardConfig` CRD does not exist until after the DSC reconciles:

```
oc apply -f manifests/04-rhoai-config/dscinitialization.yaml
oc wait --for=jsonpath='{.status.phase}'=Ready \
  dscinitialization/default-dsci --timeout=600s

oc apply -f manifests/04-rhoai-config/datasciencecluster.yaml
oc wait --for=jsonpath='{.status.phase}'=Ready \
  datasciencecluster/default-dsc --timeout=600s

oc apply -f manifests/04-rhoai-config/odh-dashboard-config.yaml
```

Note: The RHOAI operator auto-creates `default-dsci` on install. The `oc apply` over it produces a harmless annotation warning on the first run.

---

**6. Deploy a Model**

Create and label the `llm` namespace:

```
oc create namespace llm
oc label namespace llm opendatahub.io/generated-namespace=true \
  maas.opendatahub.io/gateway-access=true \
  opendatahub.io/dashboard=true --overwrite
```

Deploy the simulator (CPU only, no GPU required — best for first-time validation):

```
oc apply -k manifests/05-maas-models/simulator/
```

For GPU models (label the namespace first, then choose one):

```
# Granite 4.0-h-tiny FP8 — ~8 GB VRAM, any NVIDIA GPU
oc apply -k manifests/05-maas-models/granite-tiny-gpu/

# Gemma 2 9B IT FP8 — ~12 GB VRAM, single A10G
oc apply -k manifests/05-maas-models/gemma/

# GPT-oss-20b — ~16 GB VRAM, A10G or better (g6e.2xlarge = L40S works)
oc apply -k manifests/05-maas-models/gpt-oss-20b/
```

Check readiness:

```
oc get llminferenceservice -n llm
oc get maasmodelref -n llm -o wide
oc get maasauthpolicy,maassubscription -n models-as-a-service
```

---

**7. Verify**

```
./manifests/06-verification/verify.sh
```

Manual health check:

```
CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
curl -sk "https://maas.${CLUSTER_DOMAIN}/maas-api/health"
# Expected: {"status":"healthy"}
```

Test inference with an API key:

```
API_KEY=$(curl -sk -X POST \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"name":"test","expiresIn":"1h","subscription":"simulator-free"}' \
  "https://maas.${CLUSTER_DOMAIN}/maas-api/v1/api-keys" | jq -r '.key')

curl -sk "https://maas.${CLUSTER_DOMAIN}/v1/models" \
  -H "Authorization: Bearer ${API_KEY}"
```

---

### Troubleshooting

- Gateway pod CrashLoopBackOff after model deploy → Wasm OOMKill; ensure `gateway-resources.yaml` was applied (Step 3) so the `2Gi` memory limit is set via `parametersRef`.
- `Kuadrant MissingDependency` → restart the kuadrant-operator pod, then re-wait.
- Simulator stuck in `Init:0/1` → HuggingFace Xet hang; patch `HF_HUB_DISABLE_XET=1` on the deployment.
- Dashboard shows "Not Ready" with CPU limit → cluster-specific resource pressure; check pod events in `redhat-ods-applications`.
- DSCI/DSC annotation warning on first apply → harmless; re-apply is clean.

---

### Cleanup

```
oc delete -k manifests/05-maas-models/simulator/
# Full teardown: see Phase 9 of the companion lab guide
```

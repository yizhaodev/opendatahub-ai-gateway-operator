# Architecture: opendatahub-operator + ai-gateway-operator

This document describes how the ODH platform operator and the AI Gateway module operator work together to deploy the batch-gateway stack.

## 1. Overview

The system is a three-layer operator hierarchy:

```
User
 │
 │  creates DataScienceCluster CR
 ▼
┌──────────────────────────────────────────────────┐
│  opendatahub-operator  (platform operator)       │
│                                                  │
│  Modules controller watches DSC, for each        │
│  enabled module:                                 │
│   1. Renders the module's Helm chart             │
│   2. Deploys module operator + CRD               │
│   3. Creates the module CR (e.g. AIGateway)      │
│   4. Reads module CR status for DSC aggregation  │
└──────────────┬───────────────────────────────────┘
               │  Helm install → Deployment, RBAC, CRD
               │  SSA apply   → AIGateway CR
               ▼
┌──────────────────────────────────────────────────┐
│  ai-gateway-operator  (module operator)          │
│                                                  │
│  Watches AIGateway CR, for each managed          │
│  sub-component:                                  │
│   1. Renders kustomize manifests                 │
│   2. Deploys sub-component via SSA               │
│   3. Reports status back on AIGateway CR         │
└──────────────┬───────────────────────────────────┘
               │  kustomize render + SSA
               ▼
┌──────────────────────────────────────────────────┐
│  batch-gateway-operator  (sub-component)         │
│                                                  │
│  Watches LLMBatchGateway CR, manages actual      │
│  batch inference gateway workloads               │
└──────────────────────────────────────────────────┘
```

## 2. Build process

### 2.1 Each sub-component prepares its manifests

Each sub-component operator (e.g. batch-gateway-operator) lives in its own upstream repo and provides a standard kustomize layout under its `config/` directory, including:
- **CRD** (`crd/bases/`) — the custom resource the sub-component operator watches (e.g. `LLMBatchGateway`).
- **Manager** (`manager/`) — the Deployment for the sub-component operator.
- **RBAC** (`rbac/`) — ClusterRole, ClusterRoleBinding, ServiceAccount, leader election role.
- **Overlays** (`overlays/odh/`, `overlays/rhoai/`) — platform-specific kustomize overlays for ODH and RHOAI.

### 2.2 ai-gateway-operator generates RBAC and Helm chart

- Run `make get-manifests` (`hack/scripts/get-manifests.sh`) will fetch the repo's kustomize manifests at a pinned commit SHA and copies them into `config/manifests/batchgateway/`. These files are checked in and used at runtime by the controller. Upgrading a sub-component is an explicit "bump SHA" commit in `get-manifests.sh`.

- Run `make manifests` generates `config/rbac/role.yaml` from these markers.

- Run `make helm` (`cmd/chartgen`), which reads the kustomize overlay (`config/default/`) and picks up `config/rbac/role.yaml`, CRDs, Deployment, ConfigMap, etc. The output is written to `config/chart/`, checked in, and consumed directly by opendatahub-operator.

### 2.3 opendatahub-operator consumes the Helm chart

opendatahub-operator's `get_all_manifests.sh` downloads each module's Helm chart from its repo at a pinned commit SHA (configured in `ODH_COMPONENT_CHARTS` / `RHOAI_COMPONENT_CHARTS` maps). The downloaded charts are bundled into the opendatahub-operator container image at `/opt/charts/`. At runtime, the modules controller reads charts from this path (`DEFAULT_CHARTS_PATH=/opt/charts`) to render and deploy module operators via SSA.

## 3. Reconciliation flow

The following walkthrough uses batch-gateway as an example sub-component to illustrate the end-to-end reconciliation flow.

### 3.1 User creates a DataScienceCluster CR
1. User creates a `DataScienceCluster` (DSC) CR with **aigateway** set to `Managed`.

```yaml
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    aigateway:
      managementState: Managed
```

### 3.2 opendatahub-operator → ai-gateway-operator
2. opendatahub-operator watches the `DataScienceCluster` CR and sees `aigateway` set to `Managed`.
3. opendatahub-operator renders the ai-gateway-operator Helm chart (from `config/chart/`) and deploys it via SSA:

```bash
$ oc get deployment -n opendatahub -l app.kubernetes.io/name=opendatahub-ai-gateway-operator
NAME                               READY   UP-TO-DATE   AVAILABLE
opendatahub-ai-gateway-operator    1/1     1            1
```

4. opendatahub-operator creates the `AIGateway` CR with `batchGateway` set to `Managed`:

```yaml
apiVersion: components.platform.opendatahub.io/v1alpha1
kind: AIGateway
metadata:
  name: default-aigateway
spec:
  batchGateway:
    managementState: Managed
```

### 3.3 ai-gateway-operator → sub-component operators
5. ai-gateway-operator's controller watches the `AIGateway` CR.
6. ai-gateway-operator reads the spec (e.g. `batchGateway.managementState: Managed`), renders `config/manifests/batchgateway/` via kustomize, and deploys the resources via SSA:

```bash
$ oc get deployment -n opendatahub -l app.kubernetes.io/name=batch-gateway-operator
NAME                                        READY   UP-TO-DATE   AVAILABLE
batch-gateway-operator-controller-manager   1/1     1            1
```

7. ai-gateway-operator updates `AIGateway` CR status (phase: Ready, conditions, etc.).
8. batch-gateway-operator starts running and watches `LLMBatchGateway` CRD.

### 3.4 sub-component operators → workload
9. Users create the `LLMBatchGateway` CR to provision actual workloads.
10. batch-gateway-operator watches `LLMBatchGateway` CR and deploys batch-gateway workloads.


## 5. References

- [FeatureRefinement - RHAISTRAT-1064 - Implement Modular Architecture for ODH Operator](https://docs.google.com/document/d/1qGvaUsioOXl1MPm0TqSxaYR6booRyDLxz_-wTYVF8hM/edit?tab=t.3mrf1syv46a)
- [Onboarding Guide for ODH Operator Modules](https://docs.google.com/document/d/1FgN_U-6XH8M-Mu6XNeldUlTPsnw7UyPCWg5NVJJdYnw/edit?usp=sharing)
- [opendatahub-module-operator](https://github.com/lburgazzoli/opendatahub-module-operator)

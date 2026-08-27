# Component Validation Report

**Validation date:** August 27, 2026  
**Cluster:** Rancher `local` cluster  
**Platform:** RKE2 `v1.35.6+rke2r1` 

## 1. Executive Summary

The cluster was active and the component stack was generally installed and operational. The principal GitOps applications were `Synced` and `Healthy`, the relevant controllers and workloads were Ready, all seven cluster nodes were Ready, and the RKE2 Canal networking DaemonSet was fully available.

Two findings require follow-up:

1. `aiiaas/quay-oauth-token-es` was not syncing. External Secrets Operator reported that the Vault record at `secret/data/quay/oauth-credentials` did not contain the required `client-id` property. This can affect Quay OAuth integration.
2. The active Quay application pod was Ready and Running but showed 158 restarts, with the most recent restart approximately 28 minutes before the check. This is not a current availability failure, but the restart frequency is high enough to investigate.

The `app-of-apps-deployment` label could not be mapped to a currently named Kubernetes Application or TaskRun with the same literal name. Its expected outcome was nevertheless visible: Argo CD was operational and the downstream Henry-owned GitOps applications were reconciled successfully. This is therefore recorded as indirectly validated rather than independently proven.

## 2. Scope

The component names were taken from the provided ownership screenshots. Only entries owned by Henry Li were included:

- `acm-eso`
- `acm-ocp-truststore`
- `eso-secret-store`
- `managed-cluster-set`
- `upgrade-discovery`
- `deploy-cluster-networking`
- `app-of-apps-deployment`
- `ESO`
- `ArgoCD`
- `Quay and Clair`

`acm-eso`, `ESO`, and `eso-secret-store` are related parts of the same external-secret capability, but they are kept separate in the component results because they appeared as separate ownership entries.

## 3. How the Validation Was Performed

### 3.1 Rancher UI path

1. Open Rancher and select the `local` cluster.
2. Change the namespace selector at the top of the page to **All Namespaces**.
3. Use the following Rancher pages for visual inspection:

| Validation target | Rancher navigation |
|---|---|
| Nodes | `Cluster -> Nodes` |
| Deployments and Pods | `Workloads -> Deployments` and `Workloads -> Pods` |
| Services | `Service Discovery -> Services` |
| Events | `More Resources -> Core -> Events` |
| Argo CD Applications | `More Resources -> argoproj.io -> Applications` |
| ExternalSecret resources | `More Resources -> external-secrets.io -> ExternalSecrets` |
| SecretStore resources | `More Resources -> external-secrets.io -> SecretStores` or `ClusterSecretStores` |
| ManagedClusterSet | `More Resources`, then search for `ManagedClusterSet` |

The Rancher Kubectl Shell is opened using the `>_` icon in the upper-right corner. Commands were entered only after the shell showed `Connected`.

### 3.2 Validation layers

The checks were performed at several layers:

1. **Cluster baseline:** Rancher cluster state, node readiness, Kubernetes version, and CNI availability.
2. **Installation evidence:** existence of the expected Argo CD Applications, controllers, Deployments, StatefulSets, DaemonSets, and custom resources.
3. **GitOps reconciliation:** Argo CD `sync` and `health` conditions.
4. **Runtime health:** desired versus available replicas, Pod phase, container readiness, restart counts, and recent events.
5. **Functional resource health:** ExternalSecret and SecretStore readiness, ManagedClusterSet existence, and networking DaemonSet coverage.
6. **Installer history:** Tekton TaskRun results for the cluster-networking installation step.

## 4. Cluster Baseline Results

### 4.1 Rancher summary

Observed Rancher state:

- Cluster state: `Active`
- Distribution: RKE2
- Kubernetes version: `v1.35.6+rke2r1`
- Architecture: `amd64`
- Capacity shown by Rancher: 96 CPU cores and 374 GiB memory

### 4.2 Node readiness

Command:

```bash
kubectl get nodes
```

Observed result:

- Seven nodes were present.
- All seven nodes reported `Ready`.
- One node, `rancher-sles16-5-oss-installer-alpha-worker5.dev.fyre.ibm.com`, reported `Ready,SchedulingDisabled`.
- All nodes were on `v1.35.6+rke2r1`.

Interpretation:

The node layer was healthy. `SchedulingDisabled` means worker5 was cordoned. A cordoned node can remain healthy but will not accept newly scheduled workloads; the platform owner should confirm that this is intentional maintenance state.

### 4.3 Cluster networking

Command:

```bash
kubectl get daemonset -A --no-headers | grep -Ei 'canal|calico|cilium|flannel'
```

Observed result:

```text
kube-system   rke2-canal   7   7   7   7   ...   23d
```

Interpretation:

RKE2 Canal was scheduled and Ready on all seven nodes. Combined with the Ready node state, this is strong evidence that the current cluster-networking deployment is operational.

## 5. Component Results

| Component | Purpose | Installation evidence | Runtime/functional evidence | Result |
|---|---|---|---|---|
| `acm-eso` | GitOps packaging for External Secrets Operator | Argo Application `acm-eso-local` existed and was `Synced/Healthy` | ESO controller, certificate controller, and webhook were Ready | PASS |
| `ESO` | Synchronizes secret material from external providers such as Vault into Kubernetes Secrets | External Secrets namespace and controller workloads existed | Relevant controller Pods were Running and container-ready | PASS |
| `eso-secret-store` | Defines ESO connectivity and authentication to Vault | Argo Application `eso-secret-store-local` was `Synced/Healthy` | Both observed ClusterSecretStores were `Valid`, `ReadWrite`, and `Ready=True`; one dependent ExternalSecret was failing | PASS WITH WARNING |
| `acm-ocp-truststore` | Manages cluster truststore/CA configuration through GitOps | Argo Application `acm-ocp-truststore-local` was `Synced/Healthy` | No dedicated long-running workload was expected or found by the name search | PASS at GitOps level |
| `managed-cluster-set` | Defines ACM managed-cluster grouping and governance boundaries | Argo Application `managed-cluster-set-local` was `Synced/Healthy` | `acm-managed-cluster-set` existed, along with other ManagedClusterSet resources | PASS |
| `upgrade-discovery` | Discovers or publishes upgrade information for managed platform components | Argo Application `upgrade-discovery-local` was `Synced/Healthy` | No unhealthy named workload was found | PASS |
| `deploy-cluster-networking` | Installer/bootstrap step that deploys or configures cluster networking | Argo Application `cluster-networking` was `Synced/Healthy`; later Tekton TaskRuns succeeded | `rke2-canal` was fully Ready on all seven nodes | PASS |
| `app-of-apps-deployment` | Bootstraps the Argo CD app-of-apps hierarchy | No current Application or TaskRun with this exact literal name was found | Argo CD was operational and the downstream Henry-owned Applications were reconciled | INDIRECTLY VALIDATED |
| `ArgoCD` | Continuously reconciles Git declarations with cluster resources | Argo CD operator and controller resources existed | Controllers, repo server, API server, ApplicationSet, and Redis HA were Ready | PASS |
| `Quay and Clair` | Provides an image registry and vulnerability scanning | Quay operator, Quay, Clair, database, Redis, and object-storage resources existed | Workloads were Running and Ready, but the Quay app showed 158 restarts and the Quay OAuth ExternalSecret was failing | PASS WITH WARNINGS |

## 6. Detailed Evidence

### 6.1 GitOps Applications

Command:

```bash
kubectl get applications.argoproj.io -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision \
  --no-headers |
awk '$2 ~ /eso|external|trust|managed-cluster|upgrade-discovery|cluster-networking|app-of-apps|argocd|quay|clair/ {print}'
```

Observed Henry-related results:

```text
argocd   acm-eso-local                 Synced   Healthy   0.3.0
argocd   acm-ocp-truststore-local      Synced   Healthy   0.3.0
argocd   cluster-networking            Synced   Healthy   0.3.0
argocd   eso-secret-store-local        Synced   Healthy   0.3.0
argocd   managed-cluster-set-local     Synced   Healthy   0.3.0
argocd   upgrade-discovery-local       Synced   Healthy   0.3.0
```

Interpretation:

- `Synced` means the live resource state matched the desired Git state known to Argo CD.
- `Healthy` means Argo CD considered the application's managed resources operational.
- GitOps health is necessary installation evidence, but it was supplemented with workload and custom-resource checks instead of being treated as sufficient by itself.

### 6.2 Relevant workloads

Command:

```bash
kubectl get deploy,sts,ds -A --no-headers |
grep -Ei '(^|[- ])(eso|external|truststore|managed-cluster|upgrade-discovery|cluster-networking|app-of-apps|argocd|argo-cd|quay|clair)([- ]|$)'
```

Observed ready workloads included:

#### Argo CD

- `argocd-operator/controller-manager`
- `argocd/argocd-cluster-applicationset-controller`
- `argocd/argocd-cluster-redis-ha-haproxy`
- `argocd/argocd-cluster-repo-server`
- `argocd/argocd-cluster-server`
- `argocd/argocd-cluster-redis-ha-server`

All reported their desired replicas Ready and available.

#### External Secrets Operator

- `external-secrets/external-secrets`
- `external-secrets/external-secrets-cert-controller`
- `external-secrets/external-secrets-webhook`
- `quay-operator-system/quay-operator-quay-operator`

All reported Ready workloads.

#### Quay and Clair

- `quay/quay-clair-app`
- `quay/quay-clair-postgres`
- `quay/quay-quay-app`
- `quay/quay-quay-database`
- `quay/quay-quay-redis`
- `rook-ceph/rook-ceph-rgw-quay-store-a`

All reported their desired replicas Ready and available during the deployment check.

### 6.3 Pod state and restart counts

Command used for the principal runtime namespaces:

```bash
for namespace in argocd external-secrets quay quay-operator-system; do
  echo "===${namespace}"
  kubectl get pods -n "${namespace}" --no-headers \
    -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,PHASE:.status.phase,RESTARTS:.status.containerStatuses[*].restartCount,AGE:.metadata.creationTimestamp
done
```

Observed result:

- Argo CD Pods were Running and Ready.
- External Secrets Pods were Running and Ready with zero restarts at the time of inspection.
- Quay and Clair Pods were Running and Ready.
- One Clair pod showed five restarts; its last restart was approximately six days earlier.
- The Quay operator pod showed three restarts; its last restart was approximately 38 hours earlier.
- The Quay application pod showed 158 restarts; its most recent restart was approximately 28 minutes earlier.

Interpretation:

An old, stable restart count is not automatically a current health failure. The Quay application's 158 restarts and recent restart time are materially different: the Pod was currently healthy, but the history suggests a recurring process, probe, configuration, or dependency problem and merits log review.

### 6.4 External Secrets and Vault integration

Commands:

```bash
kubectl get externalsecrets.external-secrets.io -A
kubectl get secretstores.external-secrets.io -A
kubectl get clustersecretstores.external-secrets.io
```

Observed ClusterSecretStores:

```text
eso-secret-store-vault-k8s-secret      Valid   ReadWrite   True
sovcloud-vault-cluster-secret-store    Valid   ReadWrite   True
```

Most observed ExternalSecret resources reported:

```text
SecretSynced   True
```

Exception:

```text
Namespace:   aiiaas
Name:        quay-oauth-token-es
Condition:   SecretSyncedError
Ready:       False
```

Diagnostic command:

```bash
kubectl -n aiiaas describe externalsecret quay-oauth-token-es
```

Observed event:

```text
UpdateFailed ... error processing spec.data[1]
(key: secret/data/quay/oauth-credentials),
err: cannot find secret data for key: "client-id"
```

The warning had repeated 248 times over approximately 28 hours at the time of validation.

Interpretation:

The ESO platform component and Vault SecretStore connectivity were operational because the stores were Ready and many other ExternalSecrets were synchronizing. This individual resource failed because the expected property was absent or mismatched in the Vault record. The labels associated this resource with the AIaaS service-broker application, but it consumes Quay OAuth credentials and can affect Quay-related authentication functionality.

Security note: the validation did not retrieve or print Kubernetes Secret values. Commands such as `kubectl get secret <name> -o yaml` should be avoided during routine health checks because they may expose sensitive data.

### 6.5 ManagedClusterSet

Command:

```bash
kubectl get managedclusterset
```

Observed resources included:

```text
acm
default
global
ocm-sanity-clusterset
platform
```

The application-level object observed in Argo CD was `managed-cluster-set-local`, and the relevant managed-cluster-set resource was present.

Interpretation:

The GitOps object was reconciled and the ACM resource type was populated. This confirms installation and object creation. A deeper business-level check would additionally confirm that the expected ManagedClusters are bound to the intended set and that placement policies select them correctly.

### 6.6 Installer TaskRun history

Command:

```bash
kubectl get taskrun -n openshift-pipelines \
  --sort-by=.metadata.creationTimestamp \
  --no-headers |
grep -Ei 'deploy-cluster-networking|app-of-apps-deployment' |
tail -n 20
```

Observed result:

- Historical `deploy-cluster-networking` runs included `StepFailed` and `FailureIgnored` entries.
- Multiple later runs reported `True Succeeded`.
- No currently retained TaskRun with the exact literal name `app-of-apps-deployment` was returned by the filter.

Interpretation:

The historical failures did not represent the current network state because later installer runs succeeded, the cluster-networking Application was `Synced/Healthy`, all nodes were Ready, and `rke2-canal` was 7/7 Ready. The exact `app-of-apps-deployment` installer step could not be independently correlated from retained TaskRuns.

## 7. Recommended Follow-Up

### Priority 1: Repair the Quay OAuth ExternalSecret

Validate the Vault record referenced by the ExternalSecret:

```text
secret/data/quay/oauth-credentials
```

Confirm that:

- the record exists at the expected path;
- the ESO authentication role can read it;
- the required property is named exactly `client-id`;
- any other properties referenced by the ExternalSecret, such as a client secret, also exist under the expected names.

After the Vault record or manifest is corrected, confirm recovery with:

```bash
kubectl -n aiiaas get externalsecret quay-oauth-token-es
kubectl -n aiiaas describe externalsecret quay-oauth-token-es
```

The desired result is `SecretSynced=True`, with no new `UpdateFailed` events.

### Priority 2: Investigate Quay application restarts

Locate the current Quay application Pod:

```bash
kubectl -n quay get pods -o wide
```

Then inspect it:

```bash
kubectl -n quay describe pod <quay-app-pod-name>
kubectl -n quay logs <quay-app-pod-name> --all-containers --tail=200
kubectl -n quay logs <quay-app-pod-name> --all-containers --previous --tail=200
```

Review recent warnings:

```bash
kubectl get events -n quay \
  --field-selector type=Warning \
  --sort-by=.lastTimestamp
```

Look for probe failures, OOM kills, dependency timeouts, database/Redis connectivity failures, certificate errors, or OAuth/configuration initialization problems.

### Priority 3: Confirm the cordoned node is intentional

```bash
kubectl describe node rancher-sles16-5-oss-installer-alpha-worker5.dev.fyre.ibm.com
```

Confirm with the cluster administrator whether the node is intentionally cordoned. Do not uncordon it without change approval.

### Priority 4: Establish the authoritative mapping for `app-of-apps-deployment`

Confirm which retained Tekton task, Argo Application, Helm release, or repository path is intended to represent `app-of-apps-deployment`. Once mapped, add that exact resource name to future validation checks.

## 8. Reusable Read-Only Command Checklist

### Cluster and nodes

```bash
kubectl get nodes
kubectl get pods -A | grep -Ev 'Running|Completed'
kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp
```

### Henry-owned workloads

```bash
kubectl get deploy,sts,ds -A --no-headers |
grep -Ei 'eso|external|truststore|managed-cluster|upgrade-discovery|cluster-networking|app-of-apps|argocd|quay|clair'
```

### Argo CD

```bash
kubectl get applications.argoproj.io -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision

kubectl get deploy,sts,pods -n argocd
```

### External Secrets

```bash
kubectl get deploy,pods -n external-secrets
kubectl get externalsecrets.external-secrets.io -A
kubectl get secretstores.external-secrets.io -A
kubectl get clustersecretstores.external-secrets.io
```

### Quay and Clair

```bash
kubectl get deploy,pods -n quay -o wide
kubectl get pods -n quay-operator-system
kubectl get events -n quay --field-selector type=Warning --sort-by=.lastTimestamp
```

### Networking and installer history

```bash
kubectl get daemonset -A --no-headers | grep -Ei 'canal|calico|cilium|flannel'

kubectl get taskrun -n openshift-pipelines \
  --sort-by=.metadata.creationTimestamp \
  --no-headers |
grep -Ei 'deploy-cluster-networking|app-of-apps-deployment' |
tail -n 20
```

## 9. Status Definitions Used in This Report

| Status | Meaning |
|---|---|
| PASS | Installation evidence was present and the corresponding runtime or functional checks were healthy. |
| PASS WITH WARNING | The component was installed and currently available, but a meaningful operational issue requires investigation. |
| PASS at GitOps level | The Argo CD Application was reconciled and healthy, but no standalone runtime workload was expected or independently identified. |
| INDIRECTLY VALIDATED | The expected downstream outcome was healthy, but no current resource with the exact component label could be mapped. |

## 10. Limitations

- This was a point-in-time read-only health check, not a load, resilience, failover, or disaster-recovery test.
- Successful Pod readiness does not prove every end-user workflow. No test image was pushed to or pulled from Quay, no Clair scan was submitted, and no application login was attempted.
- No secret values were read or printed.
- `acm-ocp-truststore` was validated primarily through Argo CD reconciliation because no dedicated long-running workload or plainly named ConfigMap was returned by the resource-name search.
- `app-of-apps-deployment` was validated through downstream outcomes because no retained resource with the exact literal name was found.
- Restart counts and event ages are snapshots from the validation time and may change.


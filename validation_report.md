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

### 3.1 
Application:
```bash
kubectl get applications.argoproj.io -A \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,REV:.status.sync.revision \
  --no-headers |
awk '$2 ~ /eso|external|trust|managed-cluster|upgrade-discovery|cluster-networking|app-of-apps|argocd|quay|clair/ {print}'
```
```text
argocd   acm-eso-local                 Synced   Healthy   0.3.0
argocd   acm-ocp-truststore-local      Synced   Healthy   0.3.0
argocd   cluster-networking            Synced   Healthy   0.3.0
argocd   eso-secret-store-local        Synced   Healthy   0.3.0
argocd   managed-cluster-set-local     Synced   Healthy   0.3.0
argocd   upgrade-discovery-local       Synced   Healthy   0.3.0
```
### 3.2
Deployment, StatefulSet

```bash
kubectl get deploy,sts -n argocd
```
```text
NAME                                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-cluster-applicationset-controller   1/1     1            1           14d
deployment.apps/argocd-cluster-redis-ha-haproxy            3/3     3            3           15d
deployment.apps/argocd-cluster-repo-server                 2/2     2            2           15d
deployment.apps/argocd-cluster-server                      2/2     2            2           15d

NAME                                                     READY   AGE
statefulset.apps/argocd-cluster-application-controller   1/1     15d
statefulset.apps/argocd-cluster-redis-ha-server          3/3     15d
```
## 4 ESO / External Secrets
### 4.1 ESO controller
```bash
> kubectl get deploy -n external-secrets
```
```text
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
external-secrets                   1/1     1            1           15d
external-secrets-cert-controller   1/1     1            1           15d
external-secrets-webhook           1/1     1            1           15d
```
```bash
> kubectl get pods -n external-secrets
```
```text
NAME                                                READY   STATUS    RESTARTS   AGE
external-secrets-cert-controller-7cb699c7f5-q8d97   1/1     Running   0          13d
external-secrets-db5cc8dd7-kwmrs                    1/1     Running   0          13d
external-secrets-webhook-776f79b5bf-pww4q           1/1     Running   0          13d
```
### 4.2 check ExternalSecret
```bash
kubectl get externalsecrets.external-secrets.io -A
```
```text
NAMESPACE                 NAME                                    STORETYPE            STORE                                 REFRESH INTERVAL   STATUS              READY   LAST SYNC
acm-service-broker        acm-service-broker-iam-secret-es        ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    3m49s
acm-service-broker        common-service-broker-iam-secret-es     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    3m52s
acm-service-broker        pull-secret-sovereign-core-read-es      ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    3m42s
aiiaas                    aiiaas-service-broker-iam-secret-es     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSynced        True    32s
aiiaas                    common-service-broker-iam-secret-es     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSynced        True    31s
aiiaas                    iam-api-key-es                          ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSynced        True    30s
aiiaas                    quay-oauth-token-es                     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSyncedError   False   
common-service-broker     csb-config                              ClusterSecretStore   sovcloud-vault-cluster-secret-store   24h                SecretSynced        True    3h56m
concert-hub               concert-apikey-hub                      ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    92s
default                   ivia-apps-oidcp-tenant-secrets          ClusterSecretStore   sovcloud-vault-cluster-secret-store   1h0m0s             SecretSynced        True    36m
default                   ivia-msp-oidcp-tenant-secrets           ClusterSecretStore   sovcloud-vault-cluster-secret-store   1h0m0s             SecretSynced        True    37m
netbox                    netbox-config-es                        ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    2m
netbox                    netbox-pepper-es                        ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    91s
netbox                    netbox-redis-es                         ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    110s
netbox                    netbox-superuser-es                     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m                 SecretSynced        True    102s
openshift-pipelines       concert-install-admin-creds             ClusterSecretStore   sovcloud-vault-cluster-secret-store   1m                 SecretSynced        True    30s
postgres-service-broker   common-service-broker-iam-secret-es     ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSynced        True    4m22s
postgres-service-broker   postgres-service-broker-iam-secret-es   ClusterSecretStore   sovcloud-vault-cluster-secret-store   5m0s               SecretSynced        True    70s
sov-core-networking       sov-core-networking-api-secret-es       ClusterSecretStore   sovcloud-vault-cluster-secret-store   1h                 SecretSynced        True    38m
sovereign-ivia-apps       ivia-apps-oidcp-tenant-secrets          ClusterSecretStore   sovcloud-vault-cluster-secret-store   1h0m0s             SecretSynced        True    36m
sovereign-ivia-msp        ivia-msp-oidcp-tenant-secrets           ClusterSecretStore   sovcloud-vault-cluster-secret-store   1h0m0s             SecretSynced        True    56m
sovereign-ui              accountui-secret-es                     ClusterSecretStore   sovcloud-vault-cluster-secret-store   10m                SecretSynced        True    5m27s
sovereign-ui              mspui-secret-es                         ClusterSecretStore   sovcloud-vault-cluster-secret-store   10m                SecretSynced        True    5m17s
sovereign-ui              sovereign-ca-cert-es                    ClusterSecretStore   sovcloud-vault-cluster-secret-store   10m                SecretSynced        True    29s
sovereign-ui              xpm-secret-es                           ClusterSecretStore   sovcloud-vault-cluster-secret-store   10m                SecretSynced        True    5m21s
vault-aas                 es-vault-root-ca-secret                 ClusterSecretStore   sovcloud-vault-cluster-secret-store   10s                SecretSynced        True    2s
```
## 5. Quay and Clair
### 5.1 deployment and pods
```bash
> kubectl get deploy -n quay
```
```text
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
quay-clair-app        2/2     2            2           20d
quay-clair-postgres   1/1     1            1           20d
quay-quay-app         1/1     1            1           20d
quay-quay-database    1/1     1            1           20d
quay-quay-redis       1/1     1            1           20d
```

```bash
> kubectl get pods -n quay -o wide
```
```text
NAME                                  READY   STATUS    RESTARTS        AGE   IP            NODE                                                            NOMINATED NODE   READINESS GATES
quay-clair-app-5f7898bc59-pg6hx       1/1     Running   5 (6d14h ago)   14d   10.42.3.9     rancher-sles16-5-oss-installer-alpha-worker4.dev.fyre.ibm.com   <none>           <none>
quay-clair-app-5f7898bc59-qjsfr       1/1     Running   0               14d   10.42.0.213   rancher-sles16-5-oss-installer-alpha-worker1.dev.fyre.ibm.com   <none>           <none>
quay-clair-postgres-b69587bb7-kps64   1/1     Running   0               14d   10.42.0.212   rancher-sles16-5-oss-installer-alpha-worker1.dev.fyre.ibm.com   <none>           <none>
quay-quay-app-874887756-tgzqt         1/1     Running   160 (40m ago)   13d   10.42.1.4     rancher-sles16-5-oss-installer-alpha-worker2.dev.fyre.ibm.com   <none>           <none>
quay-quay-database-5cff9b9df5-2cfbv   1/1     Running   0               14d   10.42.0.211   rancher-sles16-5-oss-installer-alpha-worker1.dev.fyre.ibm.com   <none>           <none>
quay-quay-redis-86754c9749-2vgvp      1/1     Running   0               14d   10.42.0.225   rancher-sles16-5-oss-installer-alpha-worker1.dev.fyre.ibm.com   <none>           <none>
> kubectl get pods -n quay-operator-system
NAME                                           READY   STATUS    RESTARTS      AGE
quay-operator-quay-operator-84685b9c9f-7mlxf   1/1     Running   3 (42h ago)   14d
```

## 6. Cluster Networking
```bash
kubectl get daemonset -A --no-headers | grep -Ei 'canal|calico|cilium|flannel'
kube-system      rke2-canal                                 7     7     7     7     7     kubernetes.io/os=linux   23d
```

## 7. Pipeline
```bash
kubectl get taskrun -n openshift-pipelines \
>   --sort-by=.metadata.creationTimestamp \
>   --no-headers |
> grep -Ei 'deploy-cluster-networking|app-of-apps-deployment' |
> tail -n 20
sovcloud-installer-run-5j6lz-deploy-cluster-networking            False     StepFailed               2d20h   2d20h
sovcloud-installer-run-w2x9r-deploy-cluster-networking            True      Succeeded                2d19h   2d19h
sanity-vault-f4p42-deploy-cluster-networking                      False     FailureIgnored           2d19h   2d19h
sovcloud-installer-run-w2x9r-r-z9xjn-deploy-cluster-networking    True      Succeeded                2d19h   2d19h
sovcloud-installer-run-w2x9r-r-528bx-deploy-cluster-networking    True      Succeeded                2d19h   2d19h
sovcloud-installer-run-vmv78-deploy-cluster-networking            True      Succeeded                2d19h   2d19h
sovcloud-installer-run-xsvvh-deploy-cluster-networking            True      Succeeded                2d18h   2d18h
sovcloud-installer-run-rw6jt-deploy-cluster-networking            True      Succeeded                2d17h   2d17h
sovcloud-installer-run-jwg7h-deploy-cluster-networking            False     StepFailed               2d17h   2d17h
sovcloud-installer-run-jwg7h-r-5zzjd-deploy-cluster-networking    True      Succeeded                2d17h   2d17h
sovcloud-installer-run-76zfb-deploy-cluster-networking            True      Succeeded                2d16h   2d16h
sovcloud-installer-run-cr6rk-deploy-cluster-networking            True      Succeeded                2d16h   2d16h
sovcloud-installer-run-cjrgv-deploy-cluster-networking            True      Succeeded                2d16h   2d16h
sovcloud-installer-run-m6ct4-deploy-cluster-networking            True      Succeeded                2d15h   2d15h
sovcloud-installer-run-cjrgv-r-4dd9s-deploy-cluster-networking    True      Succeeded                2d15h   2d15h
sovcloud-installer-run-428jf-deploy-cluster-networking            True      Succeeded                2d15h   2d15h
sovcloud-installer-run-cjrgv-r-skxr6-deploy-cluster-networking    True      Succeeded                2d11h   2d11h
sovcloud-installer-run-cjrgv-r-jx5dw-deploy-cluster-networking    True      Succeeded                2d11h   2d11h
```



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


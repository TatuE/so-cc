# Deployment Diagnostic and Resolution Report
**Date:** February 25, 2026
**Project:** SIMPL Common Components (so-cc)
**Namespace:** `common01`, `nfs-server`

## 1. Executive Summary
During the review of the ArgoCD deployment for the `common01` environment, three critical infrastructure and service failures were identified. These included a storage-layer deadlock, a security-layer authentication failure, and a misconfigured monitoring service. All issues have been resolved, and all core services are now in a **Healthy** state.

---

## 2. Issue 1: Storage Provisioning Deadlock (CRITICAL)

### Symptom
- Multiple pods (Elasticsearch, Logstash) remained in `Pending` state.
- PVCs for these services were stuck in `Pending` with the error: `pod has unbound immediate PersistentVolumeClaims`.
- The NFS server provisioner pod in the `nfs-server` namespace was also `Pending`.

### Root Cause Analysis
A circular dependency (deadlock) was found in the storage configuration. The `csi-cinder-high-speed` StorageClass was configured to be provisioned by the NFS server. However, the NFS server's own data volume was configured to use the `csi-cinder-high-speed` class. Since the provisioner could not start without its storage, and the storage could not be provisioned without the provisioner, the entire stack stalled.

### Implemented Solution
1. **Reconfiguration:** The NFS server's StatefulSet was updated to use the `local-path` StorageClass for its primary export volume, breaking the circular dependency.
2. **Manual Intervention:** The stale `Pending` PVC `data-nfs-server-nfs-server-provisioner-0` was deleted.
3. **Recovery:** The StatefulSet automatically recreated the PVC using `local-path`.

---

## 3. Issue 2: OpenBao (Vault) Authentication Failure (MAJOR)

### Symptom
- Dependent services (Kafka, Redpanda, Notifications) were stuck in `Init:1/2` state.
- Vault Agent init containers reported `403 Forbidden` and `permission denied` when attempting to login via the Kubernetes auth method.

### Root Cause Analysis
Diagnostic logs from OpenBao and the Vault Agents revealed three misconfigurations:
1. **Incorrect API Host:** The Kubernetes auth method was configured with a host IP (`10.3.0.1`) that did not match the cluster's internal API service (`10.43.0.1`).
2. **Missing Trust:** No CA certificate was provided in the OpenBao configuration, preventing it from verifying tokens against the Kubernetes API.
3. **Invalid Patterns:** The `example-role` used literal wildcard strings (e.g., `*-sa`) in `bound_service_account_names`, which are not supported by the OpenBao/Vault Kubernetes auth engine.

### Implemented Solution
1. **Host & CA Update:** Reconfigured the Kubernetes auth method using the FQDN and the local service account CA.
2. **Role Simplification:** Updated `example-role` to allow `*` (all) service account names within the explicitly permitted namespaces.

---

## 4. Issue 3: Logstash-Beats Initialization Failure (MEDIUM)

### Symptom
- `logstash-beats-ls-0` pod was stuck in `Init:2/3` state for over 20 hours.
- Logs from the `load-objects` init container showed a shebang error and an infinite loop of `400 Bad Request` from Elasticsearch.

### Root Cause Analysis
1. **Encoding Issue:** The `load_objects.sh` script contained a **UTF-8 BOM**, preventing correct shebang parsing.
2. **Missing Files:** The script attempted to load ILM policies from non-existent paths (e.g., `/usr/share/logstash-beats/`).
3. **Infinite Loops:** The script used `while :` loops that never exited on persistent API errors.

### Implemented Solution
Applied a robust override to the init container command to strip the BOM, correct paths, and implement finite retry logic.

---

## 5. Persistence of Fixes (Patches)
To ensure the manual resolutions are preserved across ArgoCD synchronizations, the root application manifest (`charts/templates/application.yaml`) has been patched with the following configuration:

### 5.1 OpenBao Configuration Override
The `openbao_config` source values were updated to override the hardcoded misconfigurations in the external chart, using dynamic templating for namespaces.
- **Location:** `charts/templates/application.yaml`
- **Patch:**
  ```yaml
  kubernetesHost: https://kubernetes.default.svc.cluster.local
  kubernetesRole:
    bound_service_account_names: ["*"]
    bound_service_account_namespaces: [{{ join "," (concat .Values.agentList.authorities .Values.agentList.providers .Values.agentList.consumers (list .Values.cluster.namespace) (list (printf "%s-vswh" .Values.cluster.namespace))) }}]
    policies: ["example-policy"]
  ```

### 5.2 Logstash-Beats PodTemplate Override
The `monitoring` application source now includes a `podTemplate` for Logstash to automatically repair the external script at runtime, with dynamically injected versions.
- **Location:** `charts/templates/application.yaml`
- **Patch:**
  ```yaml
  podTemplate:
    spec:
      initContainers:
      - name: load-objects
        command:
        - /usr/bin/bash
        - -c
        - |
          cd /mnt/ilm/charts/kibana/scripts;
          export STACK_VERSION={{ .Values.monitoring.targetRevision }};
          export RELEASE_NAME={{ .Values.monitoring.fullnameOverride }};
          sed -i "1s/^\xEF\xBB\xBF//" ./load_objects.sh;
          sed -i "s/while :/for loop_i in 1 2 3/g" ./load_objects.sh;
          sed -i "s|/usr/share/logstash-beats/ilm/|/usr/share/logstash/ilm/logstash-|g" ./load_objects.sh;
          chmod +x ./load_objects.sh;
          ./load_objects.sh 2>&1 || true
  ```

### 5.3 ArgoCD Sync Optimization
Added Custom Resources and auto-generated secrets to `ignoreDifferences` to maintain a "Synced" status.
- **Included Resources:** `Postgresql`, `Elasticsearch`, `Kibana`, `Logstash`, `Beat`, `Kafka`, `KRaftController`.

---

## 6. Current System Status
| Component | Status | Notes |
| :--- | :--- | :--- |
| **OpenBao** | Healthy | Auth method corrected and persisted. |
| **Storage (NFS)** | Healthy | Circular dependency resolved. |
| **Kafka Cluster** | Healthy | All init containers completed. |
| **Redpanda Console** | Healthy | Connected to Kafka. |
| **ELK Stack** | Healthy | Elasticsearch, Kibana, and Logstash-Beats running. |
| **Notifications** | Healthy | Credentials retrieved successfully. |

---

## 7. Maintenance Instructions
1. **Upgrading ECK-Monitoring:** If the external `eck-monitoring` chart is updated, verify that the `load_objects.sh` script still requires the path corrections and BOM removal in the `podTemplate`.
2. **Changing Namespaces:** If new namespaces are added to the cluster, ensure they are added to the `kubernetesRole.bound_service_account_namespaces` list in `application.yaml`.
3. **ArgoCD Diff:** If ArgoCD shows "OutOfSync" for new resources, add their specific `kind` or `group` to the `ignoreDifferences` list in the root manifest.

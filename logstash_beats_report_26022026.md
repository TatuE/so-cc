# Deployment Diagnostic and Resolution Report: Logstash-Beats
**Date:** February 26, 2026
**Project:** SIMPL Common Components (so-cc)
**Namespace:** `common01`
**Component:** `logstash-beats`

## 1. Executive Summary
Following a validation of the common components deployment, a persistent failure was identified in the `logstash-beats` monitoring service. While most core infrastructure is healthy, the `logstash-beats-ls-0` pod has been failing to initialize for over 27 hours. This report details the root cause analysis and the applied configuration patch to restore service.

---

## 2. Issue Description: Logstash-Beats Initialization Failure (CRITICAL)

### Symptom
- Pod `logstash-beats-ls-0` remains in `Init:2/3` state indefinitely.
- Init container `load-objects` logs show an infinite loop of `400 Bad Request` errors from Elasticsearch.
- Script execution errors reported on the first line of `load_objects.sh`.

### Root Cause Analysis
Diagnostic commands revealed three critical errors in the upstream `eck-monitoring` script:
1. **UTF-8 BOM Encoding:** The script `load_objects.sh` contains a Byte Order Mark (`0xEF 0xBB 0xBF`) at the start of the file. This prevents the Linux kernel from correctly interpreting the `#!/bin/bash` shebang, leading to execution failures.
2. **Incorrect Directory Paths:** The script is hardcoded to find ILM policy JSON files in `/usr/share/logstash-beats/ilm/`. However, the container volumes are mounted at `/usr/share/logstash/ilm/`.
3. **Infinite Retry Loop:** The script uses a `while :` loop to retry API calls. Since the path error is persistent, the script never exits, causing the init container to hang forever.

---

## 3. Implemented Solution

### 3.1 Runtime Command Override
The root application manifest (`charts/templates/application.yaml`) was updated to include a `podTemplate` override for Logstash. This patch repairs the script at runtime using the following logic:
- **BOM Removal:** `sed -i "1s/^\xEF\xBB\xBF//"`
- **Loop Correction:** Replaces `while :` with a finite `for loop_i in 1 2 3` to ensure the container eventually fails or proceeds.
- **Path Correction:** `sed -i "s|/usr/share/logstash-beats/ilm/|/usr/share/logstash/ilm/logstash-|g"`

### 3.2 ArgoCD Sync Enabling
It was discovered that `Logstash` resources were listed in the `ignoreDifferences` section of the ArgoCD Application. This caused the controller to ignore the `podTemplate` changes.
- **Action:** Removed `logstash.k8s.elastic.co/Logstash` from the `ignoreDifferences` list in `charts/templates/application.yaml`.

---

## 4. Current System Status

| Component | Status | Notes |
| :--- | :--- | :--- |
| **OpenBao** | Healthy | Authentication working via Kubernetes method. |
| **Kafka Cluster** | Healthy | All brokers and controllers synchronized. |
| **logstash-single** | Healthy | Primary logging pipeline operational. |
| **logstash-beats** | **Failing** | Awaiting ArgoCD sync of the `podTemplate` patch. |
| **Elasticsearch** | Healthy | Status is `yellow` due to low-resource preset. |

---

## 5. Maintenance and Next Steps
1. **Validation:** Once the ArgoCD sync is complete, verify that the `load-objects` init container finishes successfully.
2. **Decommissioning:** Given that `logstash-single` is operational and `logstash-beats` is marked as deprecated, evaluate the possibility of removing the `logstash-beats` deployment once monitoring parity is confirmed.
3. **Upstream Fix:** Recommend that the `eck-monitoring` repository fixes the `load_objects.sh` encoding and paths to prevent these issues in future releases.

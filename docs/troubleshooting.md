# Troubleshooting Guide

Common issues and fixes when using these templates.

---

## Template 01 — Docker Build & Deploy

### "Cannot connect to the Docker daemon"

**Cause:** Docker-in-Docker (DinD) service is not running or TLS misconfiguration.

**Fix:**
```yaml
services:
  - docker:24.0.5-dind
variables:
  DOCKER_TLS_CERTDIR: ""    # Disable TLS for DinD
  DOCKER_HOST: "tcp://docker:2375"  # Add this if missing
```

### "Error: unauthorized: authentication required" on push

**Cause:** Registry credentials not configured or expired.

**Fix:**
1. Check that `CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD` are set (auto-set on GitLab.com)
2. For self-hosted GitLab, ensure the Container Registry is enabled: Admin → Settings → CI/CD → Container Registry
3. For ECR/GCR, verify your credential helper is configured correctly

### Slow Docker builds

**Cause:** No layer caching between pipeline runs.

**Fix:** The template uses `--cache-from` to pull the latest image as a cache source. Make sure:
1. The `latest` tag exists in your registry (push it on the first build)
2. `DOCKER_BUILDKIT=1` is set (enables BuildKit inline caching)
3. Your Dockerfile is ordered correctly — put rarely changing layers first

---

## Template 02 — Terraform Plan + Apply

### "Error: Backend initialization required"

**Cause:** Backend configuration variables not set or incorrect.

**Fix:**
```bash
# Check these CI/CD variables are set:
# AWS: TF_BACKEND_BUCKET, TF_BACKEND_KEY, TF_BACKEND_REGION
# GCP: TF_BACKEND_BUCKET, TF_BACKEND_PREFIX
```

### "Error: No configuration files"

**Cause:** `TF_ROOT` variable points to wrong directory.

**Fix:**
```yaml
variables:
  TF_ROOT: "${CI_PROJECT_DIR}/infrastructure"  # Adjust to your actual path
```

### Plan shows changes but nothing changed

**Cause:** State drift — someone made changes outside Terraform.

**Fix:**
1. Run `terraform plan` locally to confirm
2. Import the manual changes: `terraform import <resource> <id>`
3. Or refresh state: `terraform apply -refresh-only`
4. Prevent future drift with CI/CD-only access to cloud accounts

### "Error acquiring the state lock"

**Cause:** A previous pipeline run crashed without releasing the lock.

**Fix:**
```bash
# Get the lock ID from the error message, then:
terraform force-unlock LOCK_ID
```
To prevent: ensure your pipeline timeout is long enough for `terraform apply` to complete.

---

## Template 03 — K8s Helm Deployment

### "Error: UPGRADE FAILED: timed out waiting for the condition"

**Cause:** Pods are not reaching Ready state within the `--timeout` window.

**Fix:**
1. Check pod status: `kubectl get pods -n <namespace>`
2. Check pod events: `kubectl describe pod <pod-name> -n <namespace>`
3. Common reasons:
   - Image pull error (wrong tag, registry auth)
   - Readiness probe failing (wrong port, path, or app not starting)
   - Insufficient resources (CPU/memory limits too low)
4. Increase timeout if the app legitimately takes longer to start:
   ```yaml
   --timeout 10m  # increase from 5m
   ```

### "Error: rendered manifests contain a resource that already exists"

**Cause:** A resource with the same name exists but was not created by Helm.

**Fix:**
```bash
# Option 1: Adopt the existing resource
kubectl annotate <resource-type> <name> meta.helm.sh/release-name=<release> meta.helm.sh/release-namespace=<namespace>
kubectl label <resource-type> <name> app.kubernetes.io/managed-by=Helm

# Option 2: Delete the existing resource first (if safe)
kubectl delete <resource-type> <name> -n <namespace>
```

### Rollback not working

**Cause:** `--history-max` is set too low or previous revision was also broken.

**Fix:**
```bash
# List all revisions:
helm history <release-name> -n <namespace>

# Rollback to a specific working revision:
helm rollback <release-name> <revision-number> -n <namespace>
```

---

## Template 04 — Security Scanning

### "Unable to pull Trivy database"

**Cause:** Network restrictions or rate limiting on the vulnerability database download.

**Fix:**
1. Pre-cache the Trivy DB in your runner:
   ```yaml
   cache:
     key: trivy-db
     paths:
       - .cache/trivy/
   variables:
     TRIVY_CACHE_DIR: ".cache/trivy"
   ```
2. Or host a mirror of the Trivy DB internally

### "Pipeline blocked — CRITICAL vulnerability found"

**Cause:** Working as intended — Trivy found a critical CVE.

**Fix:**
1. Check `trivy-report.txt` artifact for details
2. Update the vulnerable dependency if possible
3. If it's a false positive or unfixable, create a `.trivyignore` file:
   ```
   # .trivyignore
   CVE-2024-12345  # False positive — not exploitable in our context
   ```
4. As a last resort, set `ALLOW_FAILURE_ON_VULNS: "true"` (not recommended for production)

### Scans are slow (>10 minutes)

**Cause:** Downloading the vulnerability database on every run.

**Fix:** Cache the database:
```yaml
trivy-container-scan:
  cache:
    key: trivy-db-cache
    paths:
      - .cache/trivy/
  variables:
    TRIVY_CACHE_DIR: ".cache/trivy"
```

---

## Template 05 — GCP Cloud Run

### "ERROR: (gcloud.run.deploy) PERMISSION_DENIED"

**Cause:** Service account missing required IAM roles.

**Fix:** Ensure the service account has these roles:
- `roles/run.admin` — deploy Cloud Run services
- `roles/artifactregistry.writer` — push images
- `roles/iam.serviceAccountUser` — act as the runtime service account

```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:SA_EMAIL" \
  --role="roles/run.admin"
```

### "Revision is not ready and cannot serve traffic"

**Cause:** The container is crashing on startup.

**Fix:**
1. Check Cloud Run logs: `gcloud run services logs read SERVICE_NAME --region REGION`
2. Common causes:
   - App not listening on the configured `CLOUD_RUN_PORT`
   - Missing environment variables
   - Insufficient memory (`CLOUD_RUN_MEMORY`)
3. Test locally: `docker run -p 8080:8080 YOUR_IMAGE`

### Cold start latency

**Cause:** `CLOUD_RUN_MIN_INSTANCES` is set to `0`.

**Fix:**
```yaml
variables:
  CLOUD_RUN_MIN_INSTANCES: "1"  # Keep 1 instance warm
```
Note: this incurs cost even when idle.

---

## General GitLab CI Issues

### "This job is stuck because you don't have any active runners"

**Fix:**
1. GitLab.com: Enable shared runners in Settings → CI/CD → Runners
2. Self-hosted: Register a runner: `gitlab-runner register`

### Pipeline runs on wrong branches

**Fix:** Check the `workflow: rules:` section at the top of your `.gitlab-ci.yml`. The templates use:
```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```
Adjust these rules to match your branching strategy.

### "YAML syntax error"

**Fix:**
1. Validate locally: `gitlab-ci-lint` or paste into GitLab CI Lint (CI/CD → Editor → Validate)
2. Common YAML mistakes:
   - Tabs instead of spaces (YAML requires spaces)
   - Missing quotes around strings with special characters
   - Incorrect indentation in multi-line `script:` blocks

---

## Still Stuck?

1. Check GitLab CI/CD documentation: https://docs.gitlab.com/ee/ci/
2. Read the pipeline job log — the error message usually tells you exactly what's wrong
3. Ask in the DevOps Dispatch newsletter community: [devopsdispatch.beehiiv.com](https://devopsdispatch.beehiiv.com)

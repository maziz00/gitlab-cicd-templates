# Customization Guide

How to adapt these templates to your project.

---

## Variables — The Only Things You Need to Change

Every template has a clearly marked `variables:` section at the top. In most cases, updating these values is all you need to do.

### Template 01 — Docker Build & Deploy

| Variable | Default | Change To |
|----------|---------|-----------|
| `DOCKER_REGISTRY` | `registry.gitlab.com` | Your registry (ECR, GCR, Docker Hub) |
| `IMAGE_NAME` | `$CI_REGISTRY_IMAGE` | Custom image path if needed |
| `APP_PORT` | `8080` | Your application's port |
| `CONTAINER_NAME` | `$CI_PROJECT_NAME` | Custom container name |

**Switching to ECR:**

```yaml
variables:
  DOCKER_REGISTRY: "123456789.dkr.ecr.us-east-1.amazonaws.com"
  IMAGE_NAME: "123456789.dkr.ecr.us-east-1.amazonaws.com/my-app"
```

Update `before_script` to use AWS ECR login:

```yaml
before_script:
  - apk add --no-cache aws-cli
  - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $DOCKER_REGISTRY
```

### Template 02 — Terraform Plan + Apply

| Variable | Default | Change To |
|----------|---------|-----------|
| `TF_VERSION` | `1.7.0` | Your Terraform version (pin it!) |
| `TF_ROOT` | `${CI_PROJECT_DIR}` | Path to your Terraform root module |
| `TF_WORKSPACE` | `default` | `staging`, `production`, etc. |

**Using GCS backend instead of S3:**

```yaml
variables:
  TF_CLI_ARGS_init: "-backend-config=bucket=${TF_BACKEND_BUCKET} -backend-config=prefix=${TF_BACKEND_PREFIX}"
```

### Template 03 — K8s Helm Deployment

| Variable | Default | Change To |
|----------|---------|-----------|
| `CHART_PATH` | `./charts/app` | Path to your Helm chart |
| `STAGING_NAMESPACE` | `staging` | Your staging namespace |
| `PRODUCTION_NAMESPACE` | `production` | Your production namespace |

### Template 04 — Security Scanning

| Variable | Default | Change To |
|----------|---------|-----------|
| `TRIVY_SEVERITY` | `CRITICAL,HIGH` | Add `MEDIUM` for stricter scanning |
| `ALLOW_FAILURE_ON_VULNS` | `false` | `true` to not block pipeline |

### Template 05 — GCP Cloud Run

| Variable | Default | Change To |
|----------|---------|-----------|
| `GCP_PROJECT_ID` | `your-gcp-project-id` | Your actual GCP project |
| `GCP_REGION` | `me-west1` | Your preferred region |
| `CLOUD_RUN_MEMORY` | `512Mi` | `1Gi`, `2Gi` based on app needs |
| `CLOUD_RUN_MIN_INSTANCES` | `0` | `1` to avoid cold starts |

---

## Common Customizations

### Adding Notifications (Slack)

Add this job to any template to get Slack notifications on failure:

```yaml
notify-slack-failure:
  stage: .post
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST $SLACK_WEBHOOK_URL \
        -H 'Content-type: application/json' \
        -d "{
          \"text\": \"Pipeline FAILED on ${CI_PROJECT_NAME}\",
          \"blocks\": [{
            \"type\": \"section\",
            \"text\": {
              \"type\": \"mrkdwn\",
              \"text\": \"*Pipeline Failed* :x:\n*Project:* ${CI_PROJECT_NAME}\n*Branch:* ${CI_COMMIT_REF_NAME}\n*Commit:* ${CI_COMMIT_SHORT_SHA}\n*Author:* ${GITLAB_USER_NAME}\n*<${CI_PIPELINE_URL}|View Pipeline>*\"
            }
          }]
        }"
  when: on_failure
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Adding Caching

Speed up pipelines with proper caching:

```yaml
# Node.js
cache:
  key: node-${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/

# Python
cache:
  key: pip-${CI_COMMIT_REF_SLUG}
  paths:
    - .pip-cache/

# Go
cache:
  key: go-${CI_COMMIT_REF_SLUG}
  paths:
    - .go-cache/
```

### Multi-Environment Deployment

Extend Template 03 for dev/staging/production:

```yaml
.deploy-base:
  image: alpine/helm:3.14.0
  before_script:
    - mkdir -p ~/.kube
    - echo "${KUBECONFIG}" | base64 -d > ~/.kube/config

deploy-dev:
  extends: .deploy-base
  stage: deploy-dev
  variables:
    KUBECONFIG: $KUBECONFIG_DEV
  script:
    - helm upgrade --install $CI_PROJECT_NAME-dev ${CHART_PATH} --namespace dev --set replicaCount=1
  rules:
    - if: $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH

deploy-staging:
  extends: .deploy-base
  stage: deploy-staging
  variables:
    KUBECONFIG: $KUBECONFIG_STAGING
  script:
    - helm upgrade --install $CI_PROJECT_NAME-staging ${CHART_PATH} --namespace staging --set replicaCount=2
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy-production:
  extends: .deploy-base
  stage: deploy-production
  variables:
    KUBECONFIG: $KUBECONFIG_PRODUCTION
  script:
    - helm upgrade --install $CI_PROJECT_NAME-prod ${CHART_PATH} --namespace production --set replicaCount=3
  when: manual
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

### Using OIDC Instead of Static Keys

For GCP (Template 05), replace service account keys with Workload Identity Federation:

```yaml
gcp-cloud-run-deploy:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID
  before_script:
    - |
      gcloud iam workload-identity-pools create-cred-config \
        projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID \
        --service-account=SA_EMAIL \
        --output-file=credentials.json \
        --credential-source-file=$GITLAB_OIDC_TOKEN
    - gcloud auth login --cred-file=credentials.json
```

For AWS (Template 02):

```yaml
tf-apply:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: https://gitlab.com
  variables:
    ROLE_ARN: "arn:aws:iam::123456789:role/gitlab-ci-role"
  before_script:
    - |
      STS_RESPONSE=$(aws sts assume-role-with-web-identity \
        --role-arn $ROLE_ARN \
        --web-identity-token $GITLAB_OIDC_TOKEN \
        --role-session-name "gitlab-ci-${CI_JOB_ID}")
    - export AWS_ACCESS_KEY_ID=$(echo $STS_RESPONSE | jq -r '.Credentials.AccessKeyId')
    - export AWS_SECRET_ACCESS_KEY=$(echo $STS_RESPONSE | jq -r '.Credentials.SecretAccessKey')
    - export AWS_SESSION_TOKEN=$(echo $STS_RESPONSE | jq -r '.Credentials.SessionToken')
```

---

## Combining Templates

Templates are designed to be combined. See `examples/full-pipeline-with-all-stages.yml` for a complete example that chains Build → Scan → Deploy.

Key patterns for combining:

1. Use `needs:` to create a DAG (directed acyclic graph) between jobs
2. Pass artifacts between stages (e.g., Docker image SHA from build to deploy)
3. Use `extends:` to share common configuration across jobs
4. Use `include:` to pull templates from a central repository

### Include from a Central Repo

```yaml
# In your project's .gitlab-ci.yml
include:
  - project: 'my-org/gitlab-cicd-templates'
    file: 'templates/01-docker-build-deploy.yml'
    ref: 'main'
  - project: 'my-org/gitlab-cicd-templates'
    file: 'templates/04-security-scanning.yml'
    ref: 'main'
```

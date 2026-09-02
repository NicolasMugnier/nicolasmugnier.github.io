---
layout: post
title: "Deploying a PHP Application to Kubernetes: A GitOps Walkthrough"
tags: [php, kubernetes, gitops, frankenphp, aws]
author: Nicolas Mugnier
categories: architecture
description: "How I shipped a FrankenPHP Symfony app to EKS: Terragrunt, Argo CD, non-root containers, SSM secrets, CronJobs, and probes."
image: /assets/img/deploy-php-k8s.webp
locale: en_US
---

# Deploying a PHP Application to Kubernetes: A GitOps Walkthrough

When you first ship a PHP backend to Kubernetes, the application code is rarely the hard part. What takes time, and iteration, is everything around it: the container, the pipeline, the secrets, the scheduler, and the runtime. This is the sequence of decisions I made deploying a Symfony app on FrankenPHP to EKS with a GitOps stack.

The service talks to external APIs (including an AI provider). That did not change the cluster shape. It did make secret paths, health probes, and a crash-on-boot smoke test non-negotiable.

---

## The stack

- **PHP 8.3 / Symfony**: application runtime
- **FrankenPHP**: PHP server on top of Caddy
- **Docker**: container image
- **GitHub Actions**: CI/CD
- **Terraform + Terragrunt**: AWS bootstrap
- **Argo CD + Helm**: GitOps deploy
- **AWS EKS**: cluster
- **SSM Parameter Store + External Secrets Operator**: secrets into pods

---

## 1. Bootstrapping AWS with Terragrunt

Before anything runs in the cluster, the AWS side has to exist. I used **Terragrunt** on top of Terraform: DRY includes, remote state, environment inheritance. Not a second IaC language, a wrapper.

### Remote state

State lives in S3, one key per project and environment:

```hcl
remote_state {
  backend = "s3"
  config = {
    bucket       = "<your-tfstate-bucket>"
    region       = "eu-west-3"
    key          = "${local.project}/${path_relative_to_include()}/terraform.tfstate"
    use_lockfile = true
    assume_role  = { role_arn = "<deployment-role-arn>" }
  }
}
```

`use_lockfile = true` stops two applies from corrupting the same state (another engineer, or two CI jobs).

### Application bootstrap module

A shared internal module owns the boring part: **ECR**, **IAM**, monitoring wiring. Each service does not copy that Terraform.

```hcl
inputs = {
  deployable_to = ["staging", "production"]
  monitoring = {
    enabled      = true
    environments = ["staging", "production"]
  }
}
```

---

## 2. GitOps with Argo CD and Helm

Once the AWS resources exist, Argo CD watches Git and reconciles the cluster to what the repo declares.

### Helm values hierarchy

```
deploy/argo/backend/
├── Chart.yaml              # chart
├── values.yaml             # shared defaults
├── staging/values.yaml     # staging overrides
└── production/values.yaml  # production overrides
```

A value in `values.yaml` is inherited unless an environment file overrides it. Divergence is explicit.

### Resource allocation

The first profile is intentionally small:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Requests are what the scheduler **guarantees**. Limits are the **ceiling** (CPU throttle, OOM kill on memory). Start low, raise from observed usage. Do not guess production size on day one.

---

## 3. The CI/CD pipeline

Four GitHub Actions workflows cover the lifecycle:

| Workflow | Trigger | Purpose |
|---|---|---|
| `pull-request.yml` | Every PR | Tests, PHPStan, build image, smoke-start the container |
| `default.yml` | Push to `main` | Build, push to ECR, deploy staging |
| `release_on_tag.yml` | Git tag | Deploy production |
| `infra-as-code.yml` | Changes under `deploy/terragrunt/` | Plan or apply Terraform |

### Docker smoke test in CI

After building the image on a PR, start it for a few seconds and **assert it is still running**. A `docker stop || true` alone is not a test: a crash still looks green.

```yaml
- name: Build Docker image (no push)
  uses: docker/build-push-action@v6
  with:
    push: false
    load: true
    tags: my-app-backend:test

- name: Test container stays up
  run: |
    docker run --rm -d --name test-container my-app-backend:test
    sleep 5
    docker inspect -f '{{.State.Running}}' test-container | grep -qx true
    docker stop test-container
```

This catches what PHPStan will not: a missing extension, a bad entrypoint, an env var the process needs at boot.

### Static analysis gate

Every PR runs **PHPStan** before merge. On a PHP codebase that calls external APIs, type mistakes that used to explode at runtime show up on the PR.

---

## 4. Container security hardening

The Dockerfile went through several iterations. Each change had a concrete failure behind it.

### Do not let FrankenPHP terminate TLS

FrankenPHP (Caddy) defaults to port 443 and automatic certificates. Behind a Kubernetes ingress that is the wrong split:

1. The ingress **already terminates TLS**
2. Binding 443 inside the pod wants extra privilege (`CAP_NET_BIND_SERVICE` or root)

The fix we used is a single environment variable:

```dockerfile
ENV SERVER_NAME=:80
```

`SERVER_NAME` is how Caddy sets the listen address. `:80` is plain HTTP, no automatic certs. TLS stays on the ingress.

### Drop capabilities you no longer need

FrankenPHP ships with `CAP_NET_BIND_SERVICE` on the binary so it can bind 443 as non-root. After moving off 443 and sitting behind the ingress, we dropped it:

```dockerfile
RUN setcap -r /usr/local/bin/frankenphp
```

Least privilege: keep only what the process still needs.

### Run as non-root

```dockerfile
ARG USER=www-data

RUN \
    setcap -r /usr/local/bin/frankenphp; \
    chown -R ${USER}:${USER} /config/caddy /data/caddy /app

USER ${USER}
```

The process is `www-data` (UID 33 on Debian). `chown` covers:

- `/config/caddy` and `/data/caddy`: Caddy runtime storage
- `/app`: app cache and logs

Without that, the image builds, then the process dies at runtime on `Permission denied`.

### Align UIDs between Docker and Kubernetes

Dockerfile `USER www-data` (UID 33) is not enough. The pod must declare the same UID:

```yaml
podSecurityContext:
  runAsUser: 33
  runAsGroup: 33
  fsGroup: 33
```

- `runAsUser` / `runAsGroup`: UID/GID of the process
- `fsGroup`: kubelet chowns mounted volumes to this GID at start

If these drift, volume mounts fail even when the Dockerfile is correct.

---

## 5. Secrets with External Secrets Operator

API keys live in **AWS SSM Parameter Store**. **External Secrets Operator** watches `ExternalSecret` objects, copies values into Kubernetes `Secret`s, then the chart injects them as env vars.

```yaml
externalSecrets:
  clusterSecretStore: <your-cluster-secret-store>
  params:
    - key: /param/my-project/staging/parameters/api_client_secret
      name: APP_API_CLIENT_SECRET
    - key: /param/my-project/staging/parameters/service_api_key
      name: APP_SERVICE_API_KEY
```

The failure mode I hit: Helm paths used a prefix that did not match SSM. ESO did not fail the deploy. The pod started, the env var was missing, the error showed up three layers later in application logs.

**Lesson:** grep the live SSM path before the first deploy. A wrong path is not a CI error.

---

## 6. CronJob for the scheduler

Background work is a Kubernetes **CronJob** in the same chart:

```yaml
cronJobs:
  initiate-calls:
    schedule: "0 7-15/2 * * 1-5"   # every 2h, Mon-Fri, 7am-3pm
    command: "php bin/console app:my-command"
    restartPolicy: Never
    image:
      fromApp: true
      imagePullPolicy: IfNotPresent
```

- `fromApp: true`: same image as the web deployment. Right default when `bin/console` and the HTTP server share the codebase.
- `imagePullPolicy: IfNotPresent`: skip a pull on every tick **only if the tag is immutable** (digest or a unique tag). `latest` plus `IfNotPresent` will happily run a stale image.

To disable a CronJob without deleting it, set a schedule that never fires (31 February). The Helm object stays; Kubernetes does not trigger it. Restore the real schedule to turn it back on.

---

## 7. Health probes and autoscaling

### Probes

```yaml
readinessProbe:
  failureThreshold: 3
  periodSeconds: 10

livenessProbe:
  failureThreshold: 5
  periodSeconds: 20
```

- **Readiness**: traffic only after the probe passes. 3 failures times 10s = 30s before the pod leaves the Service.
- **Liveness**: Kubernetes restarts the pod. 5 times 20s = 100s, so cache warmup does not look like a crash loop.

Both hit a dedicated endpoint (for example `/check/liveness`), not `/`.

### HPA

```yaml
hpa:
  maxReplicas: 3
  minReplicas: 1
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

At least one replica, up to three when CPU exceeds 70% or memory exceeds 80%. Staging was `maxReplicas: 1`. Staging should test the app, not the autoscaler.

---

## What I would keep doing

1. **FrankenPHP behind an ingress: `SERVER_NAME=:80`.** Default TLS on 443 fights ingress TLS.
2. **UID alignment is mandatory.** Dockerfile `USER`, `podSecurityContext`, and volume `fsGroup` must be the same UID/GID.
3. **Verify SSM paths before deploy.** A typo is silent at apply time and loud at runtime.
4. **CronJobs reuse the app image** (`fromApp: true`) in this layout.
5. **Smoke-test the image in CI** by asserting the process is still running, not by ignoring `docker stop`.
6. **Start probes loose, tighten later.** Easier than debugging restart loops on day one.
7. **Cap staging at 1 replica.** Do not rehearse HPA there.

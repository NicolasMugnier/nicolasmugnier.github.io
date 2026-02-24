# Deploying a PHP AI Application to Kubernetes: A GitOps Walkthrough

When you first ship an AI-powered backend service to production, the application code is rarely the hard part. What takes time — and iteration — is everything around it: the container, the deployment pipeline, the secrets, the scheduler, and the runtime configuration. This article walks through the real sequence of decisions made when deploying a PHP application (built on FrankenPHP) to a Kubernetes cluster using a full GitOps stack.

---

## The Stack

Before diving in, here is the toolchain involved:

- **PHP 8.3 / Symfony** — application runtime
- **FrankenPHP** — modern PHP server built on top of Caddy
- **Docker** — container image
- **GitHub Actions** — CI/CD pipelines
- **Terraform + Terragrunt** — infrastructure provisioning
- **Argo CD + Helm** — GitOps-based Kubernetes deployment
- **AWS EKS** — Kubernetes cluster
- **AWS Systems Manager Parameter Store + External Secrets Operator** — secret injection into pods

---

## 1. Bootstrapping the Infrastructure with Terragrunt

The first step before any code runs in Kubernetes is provisioning the underlying AWS infrastructure. Rather than managing raw Terraform directly, the project uses **Terragrunt** — a thin wrapper that adds DRY configuration, remote state management, and environment inheritance.

### Remote state

Terraform state is stored remotely in an S3 bucket, with a unique key per project and environment:

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

Using `use_lockfile = true` prevents concurrent Terraform runs from corrupting state — essential in a team where multiple engineers or CI jobs might plan or apply simultaneously.

### Application bootstrap module

A shared internal Terraform module handles the boilerplate: creating the **ECR repository** for Docker images, the necessary **IAM roles**, and wiring up **monitoring** (alerting channel, environments to monitor). This pattern avoids duplicating the same Terraform code across every service in the organisation.

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

Once the AWS resources exist, the application itself is deployed via **Argo CD** — a GitOps operator that watches the repository and reconciles the Kubernetes cluster state with what is declared in Git.

### Helm values hierarchy

Configuration is split across a three-level Helm values hierarchy:

```
deploy/argo/backend/
├── Chart.yaml              ← chart declaration
├── values.yaml             ← shared defaults (all envs)
├── staging/values.yaml     ← staging-specific overrides
└── production/values.yaml  ← production-specific overrides
```

This pattern avoids duplicating configuration while allowing each environment to diverge where it needs to. A value defined in `values.yaml` is automatically inherited unless overridden at a deeper level.

### Resource allocation

The base resource profile for the container is conservative and intentional:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Requests define what Kubernetes **guarantees** to the pod (used for scheduling decisions). Limits define the **maximum** it can consume before being throttled (CPU) or killed (memory). Starting low and adjusting based on observed usage is a safer approach than over-provisioning from day one.

---

## 3. The CI/CD Pipeline

Three GitHub Actions workflows handle the full lifecycle:

| Workflow | Trigger | Purpose |
|---|---|---|
| `pull-request.yml` | Every PR | Run tests, static analysis, build image |
| `default.yml` | Push to `main` | Build, push to ECR, deploy to staging |
| `release_on_tag.yml` | Git tag | Deploy to production |
| `infra-as-code.yml` | Changes to `deploy/terragrunt/` | Plan or apply Terraform |

### Docker smoke test in CI

A subtle but useful addition to the PR pipeline: after building the Docker image, the workflow starts the container for a few seconds before stopping it.

```yaml
- name: Build Docker image (no push)
  uses: docker/build-push-action@v6
  with:
    push: false
    load: true
    tags: my-app-backend:test

- name: Test container starts
  run: |
    docker run --rm -d --name test-container my-app-backend:test
    sleep 5
    docker stop test-container || true
```

This catches a class of errors that static analysis cannot: a missing PHP extension, an incorrect entrypoint, or a misconfigured environment variable that causes the process to crash immediately. The `|| true` ensures the CI step doesn't fail if the container exits cleanly on its own.

### Static analysis gate

Every PR also runs **PHPStan** before merge. This is particularly important for a PHP codebase interacting with external APIs — type errors that are only caught at runtime in a dynamic language get surfaced at PR time instead.

---

## 4. Container Security Hardening

The Dockerfile went through several iterations to reach a production-safe state. Each step addressed a specific concern.

### Switching from port 443 to port 80

FrankenPHP (built on Caddy) defaults to listening on port 443 with automatic TLS. This is elegant for standalone deployments but problematic behind a Kubernetes ingress controller, which:

1. **Terminates TLS itself** at the ingress layer — so the backend doesn't need it
2. **Requires elevated capabilities** (or root) to bind to ports below 1024

The fix is a single environment variable:

```dockerfile
ENV SERVER_NAME=:80
```

`SERVER_NAME` is Caddy's way of declaring its listening address. Setting it to `:80` switches the server to plain HTTP mode, disabling automatic certificate management entirely. TLS is handled upstream by the ingress.

### Dropping unnecessary Linux capabilities

FrankenPHP ships with `CAP_NET_BIND_SERVICE` set on its binary, allowing it to bind to privileged ports as a non-root user. Since we moved to port 80 and run behind an ingress, this capability is not needed:

```dockerfile
RUN setcap -r /usr/local/bin/frankenphp
```

Removing capabilities follows the **principle of least privilege**: the process should have exactly the permissions it needs, no more.

### Running as a non-root user

```dockerfile
ARG USER=www-data

RUN \
    setcap -r /usr/local/bin/frankenphp; \
    chown -R ${USER}:${USER} /config/caddy /data/caddy /app

USER ${USER}
```

The container runs as `www-data` (UID 33 on Debian), not root. The `chown` call covers:
- `/config/caddy` and `/data/caddy` — Caddy's runtime storage (config, certificates)
- `/app` — the application directory (cache, logs)

Without this, the process would fail to write to those directories at runtime even though the image built successfully.

### Aligning UIDs between Docker and Kubernetes

This is a common pitfall. The Dockerfile sets `USER www-data` (UID 33). The Kubernetes `podSecurityContext` must declare the same UID:

```yaml
podSecurityContext:
  runAsUser: 33
  runAsGroup: 33
  fsGroup: 33
```

- `runAsUser` / `runAsGroup` — the UID/GID the process runs as inside the pod
- `fsGroup` — Kubernetes will `chown` all mounted volumes to this GID at pod startup, ensuring the process can read and write to them

If these do not match, you get permission errors on mounted volumes even if the Dockerfile itself is correct.

---

## 5. Secrets Management with External Secrets Operator

Sensitive credentials (API keys, OAuth secrets) are stored in **AWS Systems Manager Parameter Store** and injected into pods at runtime by the **External Secrets Operator (ESO)**. ESO watches Kubernetes `ExternalSecret` resources and syncs the values from SSM into native Kubernetes `Secret` objects, which are then mounted as environment variables.

```yaml
externalSecrets:
  clusterSecretStore: <your-cluster-secret-store>
  params:
    - key: /param/my-project/staging/parameters/api_client_secret
      name: APP_API_CLIENT_SECRET
    - key: /param/my-project/staging/parameters/service_api_key
      name: APP_SERVICE_API_KEY
```

One issue encountered here: the SSM path convention. The initial paths used a prefix that did not match the actual path in Parameter Store. ESO would silently fail to fetch the secret, and the pod would start without the environment variable — causing runtime errors far from the actual root cause.

**Lesson:** Always verify SSM paths explicitly before deploying. A mismatch between the path declared in Helm and the actual parameter location in SSM will not surface as a build error.

---

## 6. Kubernetes CronJob for the Scheduler

The application needs to trigger a background task on a schedule. In Kubernetes, this is handled by a **CronJob** resource, declared through the Helm chart:

```yaml
cronJobs:
  initiate-calls:
    schedule: "0 7-15/2 * * 1-5"   # every 2h, Mon–Fri, 7am–3pm
    command: "php bin/console app:my-command"
    restartPolicy: Never
    image:
      fromApp: true
      imagePullPolicy: IfNotPresent
```

Two configuration points worth highlighting:

- `fromApp: true` — reuses the same Docker image as the main application deployment, rather than requiring a separate image build for the scheduler. This is the correct default for a monolithic PHP app where the console and the web server share the same codebase.
- `imagePullPolicy: IfNotPresent` — avoids pulling the image on every cron execution. On a self-hosted cluster with a private registry, unnecessary pulls add latency and network load. Since the image is already present on the node from the main deployment, this is safe.

A useful trick for temporarily disabling a CronJob without removing it from the configuration is to set an impossible schedule (e.g., February 31st). The Helm resource stays defined, but Kubernetes never triggers it — making it trivial to re-enable later by restoring a valid schedule.

---

## 7. Health Probes and Autoscaling

The final piece was configuring **liveness/readiness probes** and the **Horizontal Pod Autoscaler (HPA)**.

### Probes

```yaml
readinessProbe:
  failureThreshold: 3
  periodSeconds: 10

livenessProbe:
  failureThreshold: 5
  periodSeconds: 20
```

- **Readiness**: Kubernetes only routes traffic to a pod once this probe passes. 3 failures × 10s = 30 seconds of grace before the pod is marked unready and removed from the load balancer.
- **Liveness**: If a pod becomes unhealthy, Kubernetes restarts it. The higher threshold (5 × 20s = 100s) prevents restart loops during legitimate slow operations like cache warming.

Both probes hit a dedicated health check endpoint (e.g., `/check/liveness`).

### HPA

```yaml
hpa:
  maxReplicas: 3
  minReplicas: 1
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

The HPA keeps at least one replica running at all times and scales up to three when CPU exceeds 70% or memory exceeds 80%. In staging, this was capped to `maxReplicas: 1` — there is no value in testing autoscaling behaviour in a non-production environment, and it keeps the staging footprint small.

---

## Key Takeaways

Working through this deployment surfaced several patterns that apply to any PHP service on Kubernetes:

1. **FrankenPHP needs `SERVER_NAME=:80` behind an ingress.** The default TLS behaviour conflicts with ingress-level TLS termination.
2. **UID alignment is non-negotiable.** The Dockerfile `USER`, the Kubernetes `podSecurityContext`, and volume mounts must all agree on the same UID/GID.
3. **Verify secret paths before deploying.** A typo in an SSM path produces no build error — only a silent runtime failure.
4. **Use `fromApp: true` for CronJobs in a monorepo.** Reusing the application image avoids a separate build step for scheduled tasks.
5. **Start probes lenient, tighten later.** It is easier to tighten a probe threshold once you understand the application's startup and response behaviour than to debug restart loops on day one.
6. **Cap staging replicas to 1.** Staging should test code, not infrastructure scaling behaviour.


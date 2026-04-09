# CLAUDE.md — cnpe

This file provides Claude with persistent context for this repository across sessions.

## Purpose

This repo is a hands-on study environment for the **CNCF Certified Cloud Native Platform Engineer (CNPE)** exam. It contains practical examples, configurations, and experiments covering the CNCF ecosystem — organized by tool/domain.

## Exam Blueprint & Scope

The CNPE exam covers five domains. Each maps to a subdirectory in this repo.

### 1. GitOps & CD (25%) — `tekton/`, `argocd/`
Focus on how tools communicate, not just installation.

| Tool | Key Concepts |
|------|-------------|
| **Tekton** (CI) | Workspaces (sharing volumes between tasks), Task vs ClusterTask (namespace vs cluster scope), Triggers (EventListener + TriggerBinding for GitHub webhooks), Chains (supply chain security with cosign/SLSA) |
| **ArgoCD** (CD) | App-of-Apps pattern, Sync Policies (Automated Prune + Self-Heal), Argo Rollouts (Rollout object + AnalysisTemplate connecting metrics to deployments) |

### 2. Platform APIs & Self-Service (25%) — `crossplane/`, `backstage/`
Building the API that developers use.

| Tool | Key Concepts |
|------|-------------|
| **Crossplane** | Compositions (mapping a simple "Database" request to complex cloud resources), XRDs (defining developer-facing schema with constrained inputs like `size: small/large`) |
| **Backstage** | Software Templates (`template.yaml`), `fetch:template` to scaffold code, `publish:github` to create repos |

### 3. Observability & Operations (20%) — `otel/`, `opencost/`, `cilium/`
Providing visibility and cost data back to the business.

| Tool | Key Concepts |
|------|-------------|
| **OpenTelemetry** | Collector config (`otel-collector-config`): receive via OTLP, export to Prometheus and Jaeger |
| **OpenCost (FinOps)** | Cost allocation by namespace/label — answering "how much does Team A's dev environment cost?" |
| **Cilium / Hubble** | Enabling Hubble UI, using CLI to observe dropped packets (proving Network Policies work) |

### 4. Platform Architecture (15%) — `platform/`
Cluster structure and resource management.

| Concept | Key Focus |
|---------|-----------|
| **Multi-tenancy** | ResourceQuotas and LimitRanges to prevent one team from crashing the cluster |
| **Storage Classes** | Dynamic provisioning — ensuring PVCs are fulfilled automatically |

### 5. Security & Policy (15%) — `security/`
Guardrails and enforcement.

| Tool | Key Concepts |
|------|-------------|
| **Kyverno** | Writing policies that require labels on all pods (e.g., `cost-center` label enforcement) |
| **Falco** | Rules to alert on suspicious activity (e.g., shell exec into a production pod) |
| **Sealed Secrets** | Workflow: seal a local secret with the cluster's public key → commit encrypted file to Git |

## Project Structure

```
cnpe/
├── CLAUDE.md              # This file
├── devbox.json            # Dev environment tooling
├── tekton/                # GitOps & CD — Tekton CI
│   ├── README.md
│   ├── pipeline-*.yaml
│   ├── task-*.yaml
│   ├── pipelinerun-*.yaml
│   ├── event-listener*.yaml
│   ├── trigger-*.yaml
│   └── cosign.pub
├── argocd/                # GitOps & CD — ArgoCD (planned)
├── crossplane/            # Platform APIs — Crossplane (planned)
├── backstage/             # Platform APIs — Backstage (planned)
├── otel/                  # Observability — OpenTelemetry (planned)
├── opencost/              # Observability — OpenCost/FinOps (planned)
├── cilium/                # Observability — Cilium/Hubble (planned)
├── platform/              # Platform Architecture (planned)
└── security/              # Security & Policy — Kyverno, Falco, Sealed Secrets (planned)
```

Each directory has its own `README.md` with setup and usage notes.

## Tech Stack

| Tool | Domain | Purpose |
|------|--------|---------|
| **Tekton** | GitOps & CD | CI pipelines, triggers, supply chain security |
| **ArgoCD** | GitOps & CD | GitOps-based continuous delivery |
| **Argo Rollouts** | GitOps & CD | Progressive delivery (canary, blue/green) |
| **Crossplane** | Platform APIs | Infrastructure as Code via Kubernetes CRDs |
| **Backstage** | Platform APIs | Internal developer portal + software templates |
| **OpenTelemetry** | Observability | Telemetry collection and export |
| **OpenCost** | Observability | Kubernetes cost allocation (FinOps) |
| **Cilium / Hubble** | Observability | eBPF networking + network observability |
| **Kyverno** | Security | Kubernetes policy enforcement |
| **Falco** | Security | Runtime security and threat detection |
| **Sealed Secrets** | Security | Encrypted secrets safe to commit to Git |
| **Cosign** | Security | Container image signing and verification |
| **Kaniko** | CI | In-cluster container image builds |
| **Devbox** | Tooling | Reproducible dev environment (wraps Nix) |
| `kubectl` | Tooling | Kubernetes cluster interaction |
| `tkn` | Tooling | Tekton CLI |
| `tkn-pac` | Tooling | Pipelines as Code CLI |
| `jq` | Tooling | JSON processing |

## Development Commands

All tools are managed via devbox. Enter the dev shell first:

```bash
devbox shell
```

### Tekton

```bash
# Apply resources
kubectl apply -f tekton/

# Watch pipeline runs
tkn pipelinerun list
tkn pipelinerun logs --last -f
tkn pr describe --last

# Supply chain: fetch and view provenance
export PR_UID=$(tkn pr describe --last -o jsonpath='{.metadata.uid}')
tkn pr describe --last \
  -o jsonpath="{.metadata.annotations.chains\.tekton\.dev/signature-pipelinerun-$PR_UID}" \
  | base64 -d > metadata.json
cat metadata.json | jq -r '.payload' | base64 -d | jq .

# Supply chain: verify artifact signature
cosign verify-blob-attestation --insecure-ignore-tlog \
  --key k8s://tekton-chains/signing-secrets --signature metadata.json \
  --type slsaprovenance --check-claims=false /dev/null
```

### Tekton — Cluster Bootstrap

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
kubectl apply --filename https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml

# Configure Chains for local provenance storage
kubectl patch configmap chains-config -n tekton-chains \
  -p='{"data":{"artifacts.oci.storage": "", "artifacts.taskrun.format":"in-toto", "artifacts.taskrun.storage": "tekton"}}'
```

### Tekton — Pipelines as Code (PAC) Bootstrap

**Infrastructure setup (run on the k8s VM before bootstrapping):**

The kind cluster only has ClusterIP services — the Cloudflare tunnel daemon on the VM can't reach them directly. Port-forward the PAC controller to a local port that the tunnel can proxy to:

```bash
# Run on the k8s VM (keep running in background)
kubectl port-forward svc/pipelines-as-code-controller -n pipelines-as-code 8080:8080 &
```

The Cloudflare tunnel is configured to forward `https://tkn.pohome.site` → `http://localhost:8080` on the VM. This makes the PAC webhook endpoint publicly reachable by GitHub.

**Create the GitHub App:**

```bash
tkn pac bootstrap github-app \
  --github-application-name "your-app-name" \
  --route-url https://tkn.pohome.site
```

- The `--route-url` must include the `https://` scheme — GitHub rejects bare hostnames.
- PAC appends the path and registers it as the GitHub App webhook URL.
- On success, PAC creates the `pipelines-as-code-secret` in the `pipelines-as-code` namespace with the app credentials. Verify: `kubectl get secret -n pipelines-as-code pipelines-as-code-secret`

**Adding a new repository to PAC:**

1. Install the GitHub App(https://github.com/apps/lkp-tkn-pac) to the target repo — go to your GitHub App settings → "Install App" → select the repository.
2. Inside the target repository, run:

```bash
tkn pac create repository
# prompts for: repository URL, namespace to run pipelines in
```

This creates a `Repository` CR in the cluster and scaffolds `.tekton/pipelinerun.yaml` inside the repo. Commit and push that file — PAC watches for `.tekton/*.yaml` and triggers runs on matching GitHub events.

**Troubleshooting — GitHub App creation errors:**

| Error | Cause | Fix |
|-------|-------|-----|
| "Hook url is missing a scheme" | URL passed without `https://` | Re-run bootstrap with full `https://` URL |
| "Hook is invalid" / "Url must be a valid URL" | GitHub can't reach the webhook URL | Ensure port-forward is running and tunnel is active |
| `pipelines-as-code-secret` missing after bootstrap | Bootstrap failed before GitHub sent back credentials | Fix the URL/connectivity, re-run bootstrap |

### Tekton — `tkn-demo-app1` CI Pipeline

The `tkn-demo-app1` repo is wired to PAC and runs a full CI chain on every push/PR to `main`. Pipeline shape:

```
fetch-repository → lint → unit-test → sast → image-build
  (git-clone Hub)  (mvn      (mvn       (semgrep  (kaniko →
                    spotless: test)      scan)     kind-registry:5000/
                    check)                          tkn-demo-app1:{{ revision }})
```

**Reusable Tasks** (live in `cnpe/tekton/`, must be applied to the PAC pipelines namespace — `pipelines-as-code` in this cluster):

| File | Task | Notes |
|---|---|---|
| `task-kaniko.yaml` | `kaniko` | Multi-stage Kaniko build. Exposes `IMAGE_URL` + `IMAGE_DIGEST` results so cluster-wide Chains auto-signs without PipelineRun annotations. Needs `EXTRA_ARGS: ["--insecure"]` to push to the HTTP `kind-registry:5000`. |
| `task-maven-test.yaml` | `maven-test` | Generic `mvn -B $(GOALS)` runner. Reused by both `lint` (`GOALS: ["spotless:check"]`) and `unit-test` (`GOALS: ["test"]`). Image pinned to `maven:3.9.6-eclipse-temurin-21` by digest, matching the Dockerfile build stage. |
| `task-semgrep.yaml` | `semgrep` | Runs `semgrep scan --config=auto --error --metrics=off`. Uses `scan` not `ci` intentionally — `ci` requires `SEMGREP_APP_TOKEN` / Semgrep Cloud. |

**Apply (one-time, after editing any Task):**

```bash
kubectl apply -n pipelines-as-code \
  -f cnpe/tekton/task-kaniko.yaml \
  -f cnpe/tekton/task-maven-test.yaml \
  -f cnpe/tekton/task-semgrep.yaml
```

**Design decisions worth remembering for the exam:**

- **PAC namespace:** `pipelines-as-code` is both where the PAC controller runs AND where `tkn-demo-app1` PipelineRuns land. Any cluster Tasks referenced by `taskRef` must exist in this namespace.
- **`{{ revision }}` in the image tag:** PAC variable expanded to the commit SHA. Gives per-commit tags so Chains produces a distinct SLSA provenance per commit.
- **Task ordering rationale:** `lint` first (cheapest failure to produce/fix) → `unit-test` → `sast` → `image-build`. Fail as early as the failure type allows.
- **One Task, two uses:** `lint` and `unit-test` both reference `maven-test` with different `GOALS`. Follows "one concern per file" without duplicating Task definitions.
- **Separate lint Task vs Maven lifecycle binding:** Lint is a distinct Tekton task (rather than binding Spotless to the `validate` phase) so pass/fail shows up as its own node in `tkn pr describe` — better exam visibility at the cost of DX (local `mvn test` won't auto-lint).
- **DAST is intentionally NOT in this pipeline.** DAST needs a running instance and belongs post-deploy (e.g., ZAP baseline against a staging environment or an Argo CD PostSync hook). Pre-merge CI stays fast and self-contained.
- **`tkn-demo-app1/pom.xml`** has the Spotless plugin configured with Google Java Format but **no `<executions>` block** — `spotless:check` only runs when explicitly invoked. The CI calls it via the `lint` pipeline task. Local fix: `mvn spotless:apply`.

## Code Conventions

- **One concern per file**: Each YAML file defines one logical resource or a tightly coupled group.
- **Pin image digests**: Use `image@sha256:...` for all container images — required for supply chain security.
- **Descriptive resource names**: Names reflect what the resource does, not implementation details.
- **Comment non-obvious config**: Especially for Chains annotations, security contexts, workspace mappings, and policy rules.
- **File naming**: `<kind>-<name>.yaml` (e.g., `pipeline-build-push-image.yaml`, `task-hello-world.yaml`).
- **Directory per domain**: New CNCF tools get their own top-level directory with a `README.md`.

## What to Avoid

- **Don't use `latest` image tags** — breaks reproducibility and supply chain security. Always pin to a digest.
- **Don't apply resources without checking namespace context** — always confirm the target namespace first.
- **Don't commit secrets or kubeconfig files** — this repo contains study manifests only.
- **Don't add devbox packages without a clear reason** — keep tooling minimal and study-focused.
- **Don't abstract prematurely** — this is a learning repo. Explicitness over DRY; repetition is fine if it aids understanding.
- **Don't mix domain concerns across directories** — keep each tool's manifests self-contained.

## Communication Style

- **Skip preamble and pleasantries** — no "Great question!", "Sure!", or restating what was asked. Lead with the answer.
- **Always explain the reasoning** — for every suggestion or config choice, state *why* it works that way. The goal is exam understanding, not just a working artifact.
- **Concise but not shallow** — one crisp sentence of reasoning beats a paragraph of filler. If the "why" needs depth (e.g., how Chains signs a TaskRun vs PipelineRun), give it; just don't pad it.

## Notes

- `tekton/README.md` contains setup and provenance verification steps — keep it updated as study evolves.
- `cosign.pub` is the public key for verifying signed artifacts in the Tekton Chains workflow.
- `metadata.json` in `tekton/` is a generated artifact from `tkn pr describe` — not a source file.
- As each new domain is studied, add its directory, populate with working examples, and document setup in a `README.md`.
- The exam emphasizes understanding how tools communicate with each other, not just standalone configuration.

## Related Repositories

| Repo | Purpose |
|------|---------|
| **cnpe** (this repo) | Study manifests, configs, and notes for the CNPE exam |
| **tkn-demo-app1** | Sample Spring Boot (Java 21) app used as the CI target. Multi-stage `Dockerfile` (Maven build → `dhi.io/eclipse-temurin:21-alpine3.22` runtime). PAC watches `main` and runs lint → unit-test → sast → image-build on push/PR. See "Tekton — `tkn-demo-app1` CI Pipeline" above. |

# 6. Data Models and CRDs

Project Nova utilizes Kubernetes Custom Resource Definitions (CRDs) as the foundational building blocks for its execution model, enabling true GitOps-native configuration.

## Core Configuration CRDs

## `NovaJob` CRD (Multi-Repo Nexus Jobs)

In legacy CI systems, pipelines are fiercely "Repo-Centric." If you need to run an integration test that spans your `frontend-ui`, `billing-api`, and `auth-service` repositories, you usually have to write a fragile script that clones them using hardcoded SSH keys.

With Project Nova, this is solved elegantly by combining the standalone `NovaJob` Custom Resource with the strictly typed SDK to orchestrate **Multi-Repo Nexus Jobs**.

### 1. The Configuration: The `NovaJob` CRD

Instead of the pipeline code living inside one of the target repositories, we define a "Nexus" job. This can be a standalone Kubernetes Custom Resource (CR) triggered by a cron schedule, an API call, or another pipeline.

The code can be pulled from a ConfigMap, a central "platform-engineering" repo, or embedded directly in the YAML.

```yaml
apiVersion: nova.ci/v1
kind: NovaJob
metadata:
  name: nightly-integration-nexus
spec:
  type: Nexus
  trigger:
    cron: "0 2 * * *" # Runs every night at 2 AM
  # The code defines the DAG, but lives outside the target repos
  pipelineCode: |
    from nova_sdk import Pipeline, Step, Workspace

    p = Pipeline(name="Nightly-Nexus-Integration")

    @p.step(image="golang:1.21")
    def fetch_and_test():
        # Magic happens here (see SDK section below)
        pass

    if __name__ == "__main__":
        p.execute()
```

### 2. The Execution: The SDK `Workspace` API

Because Nova operates on a Zero-Trust supply chain, the developer *cannot* just run `git clone https://github.com/company/billing-api`. The system would block it.

Instead, the SDK's `Workspace` object interacts with the Coordinator to securely pull repositories that have been explicitly registered in the cluster as `NovaRepository` CRDs.

Here is how the developer writes the Python code inside that Nexus job:

```python
from nova_sdk import Pipeline, Step, Workspace

p = Pipeline(name="Nightly-Nexus-Integration")

@p.step(image="ubuntu:latest")
def multi_repo_integration():
    print("Orchestrating multi-repo environment...")
    
    # 1. Securely check out multiple registered repositories into distinct subdirectories
    # The Controller handles the Git Auth via SPIFFE seamlessly under the hood.
    billing = Workspace.checkout_repo(nova_ref="billing-api", branch="main", path="./billing")
    frontend = Workspace.checkout_repo(nova_ref="frontend-ui", branch="develop", path="./ui")
    
    # 2. Execute cross-repo integration logic
    Workspace.sh(f"docker-compose -f {billing.path}/docker-compose.yml up -d")
    Workspace.sh(f"npm run e2e-tests --prefix {frontend.path}")
```

### 3. The Audit Trail: The Graph DB Topography

This is where the "Cosmic Auditability" shines.

When the Coordinator compiles this DAG, it recognizes that this is a `Nexus` job utilizing multiple inputs. It writes the exact provenance to ArangoDB. If an auditor looks at this `PipelineRun` node, they will see multiple incoming `[:CLONED]` edges.
Beyond the `NovaJob`, Project Nova uses several other Custom Resources to manage the supply chain, event routing, and cluster security.

### `NovaRepository` CRD (Supply Chain Security)
Instead of hardcoding Git URLs in pipelines, platform engineering teams define the allowed universe of repositories globally. This ensures SLSA Level 4 compliance by preventing developers from cloning unvetted code.

```yaml
# A registered entity in the cluster
apiVersion: nova.ci/v1
kind: NovaRepository
metadata:
  name: core-app
spec:
  url: "https://github.com/org/core-app.git"
  provider: "github" # github, gitlab, bitbucket, internal
  allowedBranches: ["main", "release/*"]
  credentialsRef:
    secretName: "github-pat"
    namespace: "ci-system"
  triggers:
    - event: "push"
      branch: "main"
      job: "deploy-prod"
```

### `NovaTrigger` CRD (The Routing Engine)
Instead of placing "if/then" routing logic inside the pipeline code, `NovaTrigger` objects dictate exactly what events trigger which jobs. They support path filtering natively, giving Nova a massive "Monorepo Superpower."

```yaml
apiVersion: nova.ci/v1
kind: NovaTrigger
metadata:
  name: trigger-backend-tests
spec:
  repository: "nova-ref/backend-api"
  events: ["pull_request", "push"]
  filters:
    branches: ["main", "feature/*"]
    paths: ["src/**/*.go", "go.mod"] # Only trigger if Go files changed!
    labels: ["requires-testing"]
  action:
    targetJob: "nova-ref/backend-integration-job"
```

### `NovaPolicy` CRD (Security Governance & Backpressure)
To maintain Zero-Trust isolation, Nova heavily restricts ephemeral Agent namespaces. The `NovaPolicy` CRD defines these Default-Deny overrides and sets limits on how much telemetry an Agent can emit before the local Controller throttles it to protect the message bus.

```yaml
apiVersion: nova.ci/v1
kind: NovaPolicy
metadata:
  name: global-default-policy
spec:
  type: Global
  network:
    allow_k8s_api: false
    allow_privileged_containers: false
  telemetry:
    max_log_rate_kbps: 5000 # The Controller enforces this!
  ttls:
    maxNamespaceDuration: "2h"
    artifactRetention: "30d"
```

## Graph Data Models (`ArangoDB`)

Project Nova translates these declarative CRDs into a highly interconnected Graph Database for real-time orchestration and auditing.

### Key Nodes
- **`PipelineRun`**: Represents a single execution instance of a DAG.
- **`Stage / Step`**: Sub-units of the execution graph.
- **`Plugin`**: A gRPC adapter definition (e.g., `PostgresAdapter`).
- **`Service`**: A physical pod or external resource managed by a plugin.
- **`Artifact`**: A cryptographically sealed binary or image.
- **`User`**: OIDC-authenticated entity who triggered or reviewed a run.

### Key Edges
- **`DEPENDS_ON`**: Defines the DAG order between Steps.
- **`PRODUCED`**: Links a Step to an Artifact.
- **`CONSUMED`**: Links a Step to an Artifact (provenance).
- **`DEPLOYED_TO`**: Links an Artifact to a Stage.
- **`INTERACTED_WITH`**: Links a User to a manual sign-off node.

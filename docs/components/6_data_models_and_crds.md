# Data Models & Kubernetes CRDs

Project Nova uses Kubernetes Custom Resource Definitions (CRDs) to define the desired state of the CI/CD environment. This approach allows for native GitOps integration.

## 1. NovaRepository
The `NovaRepository` CRD is the foundation of SLSA supply chain security. It registers a source of truth for pipeline code.

```yaml
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

## 2. NovaTrigger
Standalone mapping between events (webhooks) and jobs.

```yaml
apiVersion: nova.ci/v1
kind: NovaTrigger
metadata:
  name: on-pr-opened
spec:
  repositoryRef: "core-app"
  events: ["pull_request.opened", "pull_request.synchronize"]
  job: "lint-and-test"
```

## 3. NovaPolicy
Defines the security and governance boundaries for a set of pipelines.

```yaml
apiVersion: nova.ci/v1
kind: NovaPolicy
metadata:
  name: zero-trust-default
spec:
  allowEgress: false
  allowInternet: false
  maxNamespaceDuration: "2h"
  artifactRetention: "30d"
```

## 4. Graph Data Models (ArangoDB)

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

# 4. Developer Experience (DX)

Project Nova prioritizes a seamless, GitOps-compliant Developer Experience, enabling developers to define complex pipelines in their language of choice while providing powerful local tooling.

## SDKs & Declarative Definition

Pipelines are structured as code rather than YAML, utilizing native SDKs in **TypeScript**, **Python**, and **Golang**. This approach enables type safety, linting, and unit testing of the pipeline logic itself.

*   **Decorators and Struct Tags:** Developers use language-native constructs (e.g., `@Stage`, `@Step` in TypeScript/Python, or struct tags in Go) to define the pipeline hierarchy.
*   **Logical Organization:**
    *   **Pipeline:** The top-level definition.
    *   **Stage:** A logical grouping of dependent steps (defined via `@Stage`). Stages establish clear DAG boundaries.
    *   **Step:** An individual unit of work mapping to a specific function or method (defined via `@Step`).
*   **Workspace Object:** The SDK injects a `Workspace` object into step functions. This object abstracts away the complex underlying interactions (gRPC streams, volume mounts), providing developers with a simple API to read/write artifacts, access scoped secrets, and emit metrics.

## Multi-Repo "Nexus" Jobs

Unlike legacy CI systems that are fiercely repo-centric (restricting pipelines to the repository they reside in), Nova introduces the concept of **Nexus Jobs**. 

*   **Cross-Repository Orchestration:** Developers can define a single pipeline that acts as a Nexus, orchestrating checkouts, builds, and tests across multiple different repositories within a single unified pipeline run.
*   **Secure GitOps References:** The allowed universe of cross-repositories is defined safely by platform teams via the `NovaRepository` CRD. 
*   **Unified Workspace:** The pipeline execution can pull source code from `repo-A` and integration tests from `repo-B` natively via the SDK (`workspace.checkout_repo(...)`), removing complex git-clone bash scripts.

## Matrix Testing & Dynamic Expansion

To solve the friction of testing against multiple environments or configurations, the Nova Graph Compiler supports dynamic DAG expansion at compile-time via matrix configurations.

*   **Dynamic Parallelism:** When the Coordinator parses an execution script containing matrix variables (e.g. testing against Postgres 13, 14, 15), it dynamically generates multiple parallel branches in the ArangoDB execution graph.
*   **Language-Specific Implementation:** Python and TypeScript use dynamic runtime decorators to pass matrices. Golang utilizes Struct Tags and Reflection to calculate Cartesian products and inject those constraints at compile-time.
*   **Imperative Injection:** The Nova SDK securely injects an active iteration constraint instance (`nova.Matrix`) into each unique spawned Kubernetes pod so that it executes perfectly on target.

## CLI & Validation (`nova plan`)

Before pushing code, developers can validate their pipelines locally using the Nova CLI (`nova plan`).

*   **Dry Runs:** The CLI communicates with the API Gateway to validate the pipeline definition against the current cluster state (e.g., checking if required plugins or secrets exist) without actually executing the user's workload.
*   **DAG Validation:** Compiles the SDK code locally to ensure the resulting Directed Acyclic Graph is valid and free of circular dependencies.

## Phantom Graphs

A standout feature of Nova's DX is the concept of **Phantom Graphs**.

*   **Collaborative Review:** When `nova plan` is executed, the CLI generates a temporary, shareable visualization URL (the Phantom Graph).
*   **Visual Dry-Run:** Reviewers can inspect the planned pipeline structure, evaluating resource requests and plugin dependencies before code is committed.
*   **Sticky Notes:** Team members can leave comments and annotations directly on the nodes of the Phantom Graph.
*   **Audit Lifecycle:** If the pull/merge request is approved and the code is subsequently executed, the comments attached to the Phantom Graph are permanently linked to the live Execution Graph in ArangoDB, providing crucial historical context for auditors.

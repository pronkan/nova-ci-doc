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

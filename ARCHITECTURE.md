# Project Nova Architecture

Project Nova is a cloud-native, decentralized, graph-based CI/CD orchestration platform. Its design is rooted in providing a highly secure, scalable, and developer-friendly environment for pipeline execution.

## Core Philosophy

*   **Decentralized & Graph-Backed:** Nova leverages a Graph Database (ArangoDB) to manage pipeline execution, provenance, and dependencies, offering unprecedented observability into complex workflows.
*   **Declarative Shell, Imperative Core:** Pipelines are defined declaratively via SDKs (TypeScript, Python, Go) using decorators or struct tags, but execution occurs imperatively within strictly isolated, ephemeral environments.
*   **Zero-Trust Security:** Architected on the assumption that all user code is potentially hostile. The platform enforces security through SPIFFE/SPIRE identity management, rigorous network policies (Layer 4/Layer 7), agent sandboxing, and real-time Data Loss Prevention (DLP).
*   **Hexagonal Architecture:** The Control Plane is implemented in Golang utilizing Hexagonal Architecture principles, ensuring core business logic is decoupled from infrastructure specifics (databases, message queues).
*   **Ephemeral Execution:** Every pipeline run executes in its own dedicated, isolated Kubernetes namespace (`ci-run-XXXX`), which is garbage-collected upon completion.

## System Overview

The architecture is broadly divided into two domains:

1.  **Control Plane (`ci-system` namespace):** Responsible for global orchestration, API routing, event normalization, scheduling, and telemetry processing. Components are stateless (except for ArangoDB) and communicate primarily via NATS JetStream.
2.  **Execution Layer (`ci-run-XXXX` namespaces):** Ephemeral execution units where the actual user code runs. The Execution Sandbox employs a dual-binary architecture (Controller and Agent) to strictly separate management functions from user code execution.

### Components at a Glance
*   **API & Webhook Gateway:** Handles authentication, validates incoming webhooks (HMAC), normalizes events, and publishes to the message bus.
*   **Coordinator:** The K8s operator managing event matching against CRDs (`NovaTrigger`, `NovaRepository`), DAG compilation, JIT plugin scheduling, and namespace provisioning.
*   **Telemetry Router:** A daemon that consumes logs and events from NATS, writes graph state to ArangoDB, and manages log aggregation (S3) and live UI streaming.
*   **Controller (Governor):** The local sandbox manager enforcing identity, telemetry throttling, and secret masking.
*   **Agent (Worker):** Executes user code, utilizing a Supervisor Pattern to intercept and forward logs securely via gRPC to the Controller.

For a detailed breakdown of each domain, refer to the following sub-documents:
*   [1. Control Plane](1_control_plane.md)
*   [2. Execution Layer](2_execution_layer.md)
*   [3. Security and Compliance](3_security_and_compliance.md)
*   [4. Developer Experience](4_developer_experience.md)
*   [5. Plugin Architecture](5_plugin_architecture.md)

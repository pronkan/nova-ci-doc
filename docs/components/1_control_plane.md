# 1. Control Plane Architecture

The Nova Control Plane resides in the `ci-system` Kubernetes namespace. It uses a **Hexagonal Architecture** implemented in **Golang**, isolating core scheduling and routing logic from external integrations. Communication between control plane components is event-driven, leveraging **NATS JetStream** for high-throughput, low-latency messaging.

## Components

### API & Webhook Gateway
A unified Go binary that serves as the entry point to the Nova cluster.
*   **Authentication:** Handles UI and CLI user authentication using JWT, integrating with standard OIDC providers.
*   **Webhook Validation:** Validates incoming VCS webhooks (e.g., GitHub, GitLab) using HMAC signatures.
*   **Normalization:** Translates provider-specific webhook payloads into a unified, internal `NovaEvent` structure.
*   **Publishing:** Publishes the normalized event to a NATS topic for downstream consumption by the Coordinator.

### Coordinator (Operator)
The brain of the system, designed as a highly available Kubernetes Operator managing the core state machine.
*   **High Availability:** Uses `etcd` (via standard Kubernetes leader election) to ensure only one active Coordinator processes events at a time.
*   **Event Matching:** Listens to `NovaEvent` messages and matches them against `NovaTrigger` and `NovaRepository` CRDs to determine if a pipeline execution should commence.
*   **Execution DAG Compilation:** Compiles the user's pipeline definition (from the SDK) into an execution DAG suitable for the Graph Database.
*   **JIT Plugin Scheduling:** Intelligently schedules required plugins (adapters) Just-In-Time based on step requirements and historical telemetry data (`mode="auto"` scheduling).
*   **Namespace Provisioning:** Provisions the ephemeral `ci-run-XXXX` namespaces, setting up requisite Layer 4/Layer 7 policies prior to launching the Execution Sandbox.
*   **Dry Runs (`nova plan`):** Supports \"Phantom Graph\" generation for CLI users conducting local dry-runs, allowing them to validate the DAG without running code.
*   **ChatOps Integration:** Manages pipeline timeouts, GC triggers, and routes Human-in-the-Loop interaction requests (via the `Interaction` Node).

### Telemetry Router
A stateless Go daemon responsible for observing and serializing the outcomes of pipeline runs.
*   **OTLP Consumption:** Consumes OpenTelemetry (OTLP) payloads emitted by Execution Controllers over NATS.
*   **Graph State Management:** Translates execution telemetry into state change commands, writing node statuses directly to the **ArangoDB** Graph Database.
*   **Log Flow:** Buffers raw execution logs to robust, long-term storage (e.g., S3 or deep PVCs).
*   **Live UI Streaming:** Broadcasts real-time logs and status updates to the user interface via WebSockets.

### Graph Database (ArangoDB)
The stateful core of the platform. ArangoDB was chosen for its high-performance C++ engine, multi-model capabilities (Graph + document), and robust Kubernetes Operator.
*   **Graph Storage:** Stores the execution DAG, visualizing pipeline components (Stages, Steps, Services/Plugins, Interaction Nodes).
*   **Provenance Data:** Acts as the immutable audit log for the platform, satisfying strict compliance requirements (SLSA Level 4).

### Message Bus (NATS JetStream)
Acts as the central nervous system connecting the API Gateway, Coordinator, and Telemetry Router. It ensures reliable delivery and enables decoupled scaling of the control plane microservices.

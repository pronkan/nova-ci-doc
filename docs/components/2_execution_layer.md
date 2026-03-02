# 2. Execution Layer Architecture

The Execution Layer in Nova is designed around the principle of **ephemeral, isolated sandboxing**. Each pipeline execution occurs within a uniquely generated Kubernetes namespace (e.g., `ci-run-XXXX`). 

This architecture guarantees that tenant workloads cannot cross-contaminate and that potentially hostile code is contained.

## Core Concepts

### Ephemeral Namespaces
When the Coordinator dictates that a pipeline should run, it provisions a new, temporary namespace.
*   **Isolation:** The namespace is heavily restricted using Layer 4 Kubernetes Network Policies and Layer 7 Istio Authorization Policies.
*   **Garbage Collection:** Upon pipeline completion (success, failure, or timeout), the entire namespace is destroyed, ensuring no residual state or artifacts are left behind.

### The Supervisor Pattern
To ensure robust security and reliable telemetry, user code does not run directly on the host or in an unmonitored container.
Instead, Nova employs a **Supervisor Pattern**. The Agent binary forks and executes the user's code, capturing `stdout` and `stderr` directly via operating system pipes, preventing any bypass of the logging mechanisms.

---

## Components

The Execution Sandbox relies on a dual-binary architecture: the Controller and the Agent.

### Controller (Governor)
The Controller is a Go binary deployed alongside the Agent (or as a separate pod within the same namespace) acting as the local namespace manager and security enforcer.
*   **SPIFFE/mTLS Enforcement:** Uses SPIFFE identities to secure all gRPC communication with the Control Plane and the Agent.
*   **Telemetry Throttling:** Aggregates and throttles high-volume log output from the Agent before forwarding it to the Telemetry Router, preventing denial-of-service against the Control Plane.
*   **Data Loss Prevention (DLP):** Acts as a secure courier for artifacts. It prevents data exfiltration by strictly controlling what leaves the namespace.
*   **Secret Masking (Aho-Corasick):** Implements an Aho-Corasick algorithm to scan all outbound log streams in real-time, redacting identified secrets before they ever reach the control plane or persistent storage.
*   **The "Shelve" Concept:** Manages short-lived, ephemeral artifacts passed between pipeline stages. These are stored temporarily and destroyed when the namespace is garbage-collected.

### Agent (Worker)
The Agent is the workhorse Go binary that physically executes the user's code.
*   **Execution:** Pulls the user's code/container image and runs it.
*   **Isolation:** It is restricted from accessing the K8s API, the internal Control Plane directly, or any sensitive resources.
*   **Communication:** Communicates *exclusively* via standard gRPC contracts (`controller.proto`) with the local Controller. It cannot bypass the Controller to send data outwards.

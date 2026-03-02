# 5. Plugin Architecture

Nova is designed to be extensively extensible without compromising its Zero-Trust architecture. Unlike legacy CI systems that rely on potentially vulnerable scripts or privileged containers, Nova relies on strictly typed **universal gRPC contracts**.

## Universal Contract (`plugin.proto`)

All plugins, regardless of their function or origin, communicate with the platform via a standardized `.proto` interface.

*   **Seamless Integration:** This contract defines the methods for plugin initialization, readiness checks, status updates, and teardown.
*   **Strict Adherence:** Plugins cannot bypass this contract to access the K8s API or arbitrary cluster resources. The Controller monitors all gRPC traffic to ensure compliance.

## Deployment Modes

Plugins are defined in the SDK using the `@Plugin` annotation and are mapped to `Service` nodes in the ArangoDB graph. They can operate in various modes:

*   **Sidecars:** Deployed as containers alongside the Agent within the same `ci-run-XXXX` namespace.
*   **Standalone Pods:** For heavier services that require isolation or persistence independent of the ephemeral pipeline run.
*   **Cloud Services:** Adapters that connect out to managed services (e.g., AWS RDS, GCP Spanner).

## Innovative Routing & Scheduling

### Transparent TCP Proxying
For "passive" plugins like databases (e.g., a temporary PostgreSQL instance for integration testing), the local Agent environment requires standard TCP connectivity, not gRPC.
*   **Loopback Binding:** The Nova plugin adapter transparently binds a port on the Agent's `localhost` loopback interface.
*   **Traffic Forwarding:** It proxies the raw TCP traffic over the secure gRPC/mTLS channel to where the plugin is actually running, ensuring the user's code can connect using standard client libraries without custom configuration.

### JIT `mode="auto"` Scheduling
To optimize cluster resources, developers can specify `mode="auto"` for intense plugins.
*   The Coordinator uses historical telemetry data from ArangoDB to intelligently schedule the plugin.
*   Instead of starting the plugin at the absolute beginning of the pipeline (tying up resources unnecessarily), the Coordinator schedules the plugin "Just-In-Time"—shortly before the specific Stage or Step that requires it is scheduled to execute.

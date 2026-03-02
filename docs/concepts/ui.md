# UI Concepts & Phantom Graphs

Project Nova reimagines the CI/CD user interface not just as a dashboard for logs, but as a collaborative, interactive canvas for visualizing and debugging pipeline execution.

## The Phantom Graph

Before code is ever committed or executed, developers can run `nova plan` locally. This command interacts with the Control Plane to compile the pipeline code into an execution DAG (Directed Acyclic Graph) and validate it against the cluster's current state.

Instead of just returning a wall of JSON to the CLI, `nova plan` generates a **Phantom Graph**:
*   **Visual Dry-Run:** The Phantom Graph is rendered in the Nova UI via a temporary, shareable URL. 
*   **Resource & Dependency Review:** Reviewers (e.g., Security, Platform Engineers, or peers) can visually inspect the planned pipeline structure, evaluating resource requests, plugin dependencies, and the execution order before any compute is spun up.

## Collaborative "Figma-Style" Sticky Notes

Nova brings the collaborative features of modern design tools to the CI/CD process:
*   **Interactive Canvas:** Team members can leave comments, warnings, or questions directly on individual nodes (Steps, Plugins, Artifacts) within the Phantom Graph using sticky notes.
*   **Contextual Discussions:** "Why is this integration test dropping its parallel execution?" or "Ensure this database plugin uses the updated test schema."
*   **Approval Workflows:** For nodes requiring human-in-the-loop interaction (the `Interaction` Node type), reviewers can approve or deny the step directly from the graph interface.

## Historical Attachment & Auditing

The collaborative context generated during the Phantom Graph review phase is never lost. 
*   **Lifecycle Persistence:** When a pull/merge request is approved and the pipeline code is finally executed in the cluster, all the comments, sticky notes, and review threads attached to the temporary Phantom Graph are permanently attached to the live, executing Graph DB record.
*   **Audit Trail:** This provides crucial historical context for auditors and future debuggers. If an issue occurs in production down the line, an engineer can look at the pipeline execution graph and see the exact conversations and decisions that happened *before* that pipeline was merged.

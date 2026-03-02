# Pipeline SDK Concepts: The Hybrid Compiler Model

Nova abandons traditional, clunky YAML configurations in favor of a "Declarative Shell, Imperative Core" architecture. Pipelines are defined using native SDKs in **TypeScript**, **Python**, and **Golang**.

This approach provides the type-safety and predictability of Infrastructure-as-Code (IaC) with the readable, intuitive developer experience of decorators and struct tags. Developers write pure code, and the Nova compiler handles the complex parallel execution DAG.

## Python & TypeScript: The Decorator Approach

In dynamic languages like Python and TypeScript, developers use standard standalone functions for their CI steps. Decorators define the execution graph (the "when" and "where"), while the function body defines the logic (the "how").

```python
import ci_postgres_plugin

# The Declarative Shell: K8s constraints and plugin routing
@Plugin(name="db", provider="ci-registry/postgres", mode="auto", version="15", standalone_db=True)

# Define a step and its dependencies
@Step(name="Integration_Tests", depends_on=["db"])
def run_tests(workspace):
    # The Imperative Core: Execution logic
    # The Workspace SDK abstracts away gRPC calls, secret injection, and volume mounts
    creds = ci_postgres_plugin.get_credentials("db") 
    workspace.env["DATABASE_URL"] = f"postgres://{creds.user}:{creds.password}@localhost:5432/testdb"
    
    workspace.sh("npm run test:e2e")
```

## Golang: The Struct Tag Approach

Go doesn't have runtime decorators, so the Nova Go SDK leverages **Struct Tags** and **Reflection** to build the DAG. Developers define their pipeline structure in a central struct, and attach execution logic using pointer receiver methods.

```go
package main

import (
    "context"
    "fmt"

    "github.com/nova-ci/sdk/go/ci"
    "github.com/nova-ci/plugins/postgres"
)

// The Declarative Shell: The DAG is defined via Struct Tags
type BackendPipeline struct {
    // Adapter configuration
    DB postgres.Plugin `ci:"plugin, name=db, mode=auto, version=15"`

    // Pipeline Steps
    Lint  ci.Step `ci:"step, stage=test"`
    Tests ci.Step `ci:"step, stage=test, depends_on=Lint, uses=db"`
    Build ci.Step `ci:"step, stage=build, depends_on=Tests"`
}

// The Imperative Core: Logic is bound to Step struct fields via Methods
func (p *BackendPipeline) RunTests(ctx context.Context, ws *ci.Workspace) error {
    creds := p.DB.GetCredentials()
    
    // Abstracted workspace interaction
    ws.Env["DATABASE_URL"] = fmt.Sprintf("postgres://%s:%s@localhost:5432/testdb", creds.User, creds.Pass)
    
    return ws.Sh("go test ./...")
}

func main() {
    // Using Reflection, the SDK parses the struct tags to generate the JSON DAG 
    // for the Coordinator before actual K8s pods are spun up.
    ci.Execute(&BackendPipeline{})
}
```

## Matrix Testing & Dynamic Expansion

To solve the friction of testing against multiple environments or configurations, the Nova Graph Compiler supports dynamic DAG expansion at compile-time via matrix configurations.

Instead of writing repetitive YAML blocks or bash loops, developers can pass variables (matrices) directly into their SDK decorators or struct tags.

### Dynamic Parallelism
When the Coordinator parses an execution script containing matrix variables, it dynamically generates multiple parallel branches in the ArangoDB execution graph. This translates a single `@Step` definition into *N* distinct parallel components natively rendered in the graph.

```python
# The compiler dynamically expands this into multiple parallel DAG branches
@Service(name="database", image=f"postgres:{matrix.pg_version}")

@Step(name="Run_Tests", depends_on=["database"])
def run_tests(workspace, matrix):
    # This step will spawn in parallel for every version defined in the triggering matrix
    workspace.sh("make test")
```

### Matrix Testing in Golang

While Python and TypeScript use dynamic runtime decorators for matrix expansion, Golang requires a different approach due to its statically typed and compiled nature. Because Go lacks runtime decorators, the Nova Go SDK leverages **Struct Tags and Reflection** to handle matrix expansion.

**1. The Compilation (DAG Generation):** The `nova plan` hybrid compiler reads the matrix dimensions from the struct tags, calculates the Cartesian product, and generates parallel `Step` nodes in the ArangoDB graph *before* execution.
**2. The Imperative Injection:** The Nova SDK injects a `nova.Matrix` object into the executing Go method so the isolated Kubernetes pod knows exactly which permutation it is responsible for.

```go
package main

import (
	"context"
	"fmt"
	"github.com/nova-ci/sdk-go/nova"
)

// 1. Define the Matrix Dimensions
// These fields will be expanded into a Cartesian product by the compiler.
type TestMatrix struct {
	GoVersion []string `nova:"matrix"`
	DBVersion []string `nova:"matrix"`
}

// 2. The Declarative Shell
// We define our pipeline structure and attach the matrix using struct tags.
type BackendPipeline struct {
	// The Plugin adapter (will be dynamically scheduled for each matrix branch)
	DB nova.Plugin `nova:"plugin, provider=postgres, mode=auto"`

	// We attach the matrix struct to the step
	Integration TestMatrix `nova:"step, name=integration-tests"`
}

// 3. The Imperative Core
// We use a pointer receiver to bind the execution logic.
// Notice the injected `nova.Matrix` object, which gives this specific pod its assigned variables.
func (p *BackendPipeline) IntegrationTests(ctx context.Context, ws *nova.Workspace, m nova.Matrix) error {
	// The SDK injects the specific permutation for this execution pod
	goVer := m.Get("GoVersion")
	dbVer := m.Get("DBVersion")

	ws.Log().Infof("Running matrix branch: Go %s | DB %s", goVer, dbVer)

	// Dynamically configure the plugin proxy port or environment based on the matrix variable
	dbProxyPort := p.DB.GetProxyPort(ctx, map[string]string{"version": dbVer})
	dbUrl := fmt.Sprintf("postgres://user:pass@localhost:%d/testdb", dbProxyPort)

	// Execute the code using the specific matrix variables
	cmd := fmt.Sprintf("GO_VERSION=%s DB_URL=%s make test-e2e", goVer, dbUrl)
	return ws.Exec(cmd)
}

func main() {
	// 4. The Compiler Entrypoint
	// Populates the matrix values for the dry-run compilation
	pipeline := &BackendPipeline{
		Integration: TestMatrix{
			GoVersion: []string{"1.20", "1.21"},
			DBVersion: []string{"14", "15"},
		},
	}
	
	nova.Execute(pipeline)
}
```

This capability ensures that developers can review their expanded matrix dependencies safely within the Phantom Graph dry-run before any actual pod execution begins.

## Key Benefits
1. **Type Safety & Autocomplete:** Full IDE support for plugins and workspace methods.
2. **Local DAG Validation:** Developers can catch circular dependencies and syntax errors locally via `nova plan` before waiting for a K8s pod to spin up.
3. **Implicit Dependency Resolution:** The compiler translates `@Step(depends_on=["..."])` tags into strictly ordered execution graphs within ArangoDB.

# Technical Reference: Temporal & Durable Execution in Agentic Systems

Temporal is an open-source, developer-first orchestration platform designed to guarantee **Durable Execution** [workos.com, temporal.io]. It allows developers to write standard, linear programming code (Python, Go, TypeScript, Java) that runs to completion regardless of infrastructure failures, network partitions, or transient third-party API outages [workos.com, temporal.io].

------

## 1. Core Capabilities

Temporal transforms stateless, fragile distributed applications into resilient, self-recovering systems by implementing several foundational capabilities:

- **State Reconstruction via Event Sourcing:** Temporal intercepts every boundary between your business logic (Workflow) and external operations (Activities) [workos.com, temporal.io]. It appends these boundaries as immutable events in a ledger [workos.com, temporal.io]. If the host server crashes mid-execution, Temporal re-spins the code on a new worker, "replays" the history, and reconstructs the local memory, variables, and stack trace exactly where they were—**without re-running previously executed LLM or API calls** [workos.com, temporal.io].
- **Durable Working Memory:** Unlike long-term databases (which store episodic/structural agent memories), Temporal persists the active **working memory** (active state machines, variable scopes, current planning loops) natively as standard program state [workos.com].
- **Native Resiliency Patterns:** The SDK natively wraps flaky external calls (like LLMs and tool APIs) in Activities [workos.com]. Retries, exponential backoffs, rate-limits, and circuit-breaking are declared via code and executed automatically by the cluster [workos.com, temporal.io].
- **Event-Driven Human-in-the-Loop (HITL):** Workflows can pause execution indefinitely waiting for an external event ("Signal") [workos.com, temporal.io]. While waiting, the workflow is completely dehydrated from server memory and consumes **zero compute resources** [workos.com].

------

## 2. Product Offerings

| Aspect            | Self-Hosted (Open Source)                                    | Temporal Cloud (SaaS)                                        |
| :---------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| Licensing         | MIT License (100% Free) [temporal.io]                        | Commercial Managed Service [temporal.io]                     |
| Operational Scope | You host and manage the database, cluster services, and scaling. | Temporal manages cluster scaling, state tracking, and DB clustering [mikhail.io]. |
| Compute Location  | Workers run on your own compute (K8s, VMs, Serverless) [mikhail.io]. | Workers still run on your compute (BYOC Model) for security/compliance [mikhail.io]. |
| Pricing Model     | Infrastructure costs only.                                   | Consumption-based: Actions (~$50/M) + Storage [temporal.io]. |

------

## 3. Structural Limitations

While highly powerful, Temporal is **not** a silver bullet for every engineering problem:

- **Workflow-Oriented, Not Model-Driven:** Temporal does not make intelligent decisions, perform prompt evaluations, or run neural network inferences [zylos.ai]. It is an **orchestration engine**, providing a deterministic skeleton around your non-deterministic AI "brain" [workos.com].
- **The Determinism Constraint:** Workflow code **must be 100% deterministic** [temporal.io]. Inside a Workflow, you cannot call random UUIDs, read current system time (`datetime.now()`), or execute external network requests directly [temporal.io]. All non-deterministic side-effects must be isolated inside **Activities** or evaluated using specialized SDK helpers [temporal.io].
- **History Bloat:** Event-sourcing histories are constrained. If a workflow runs an infinite agentic loop generating tens of thousands of steps, the history will bloat, slowing down replay speeds. The developer must manually call `ContinueAsNew` to reset the event history [temporal.io].

------

## 4. Infrastructure Architecture & Scaling Model

Temporal decouples the **Control Plane** (state and queues) from the **Data Plane** (your business logic/compute) [mikhail.io].

```
                  ┌──────────────────────────────────────────────┐
                  │               TEMPORAL CLUSTER               │
                  │                (Control Plane)               │
                  │                                              │
 ┌──────────┐     │  ┌───────────┐   gRPC Routing  ┌──────────┐  │
 │  Client  │ ───>│  │ Frontend  │ ──────────────> │ Matching │  │
 │ (gRPC)   │     │  │  Service  │                 │ Service  │  │
 └──────────┘     │  └─────┬─────┘                 └────┬─────┘  │
                  │        │                            ▲        │
                  │  ┌─────▼─────┐                      │ Poll   │
                  │  │  History  │ ◄────────────────────┘ (gRPC) │
                  │  │  Service  │                               │
                  │  └─────┬─────┘                               │
                  │        │                                     │
                  │  ┌─────▼─────┐                               │
                  │  │ Database  │                               │
                  │  └───────────┘                               │
                  └──────────────────────────────────────────────┘
                                  ▲
                                  │ Poll & Execute
                                  │ (mTLS gRPC)
                                  │
                  ┌───────────────┴──────────────┐
                  │         YOUR COMPUTE         │
                  │         (Data Plane)         │
                  │                              │
                  │   ┌──────────────────────┐   │
                  │   │    Worker Process    │   │
                  │   │ (Workflows & Acts)   │   │
                  │   └──────────────────────┘   │
                  └──────────────────────────────┘
```

### Major Infrastructure Components:

1. **Frontend Service (Stateless):** The gateway of the cluster. It rate-limits, authenticates (via mTLS), and routes incoming gRPC client requests to the correct History or Matching shard.
2. **History Service (Stateful & Partitioned):** The core transaction processor. It is split into logical partitions called **History Shards** (hashed by `WorkflowID`) [medium.com]. Only the node currently assigned Shard X can mutate state for Shard X, eliminating database locking contention [medium.com].
3. **Matching Service (Stateless):** Manages Task Queues in-memory [medium.com]. It matches waiting tasks to polling workers over long-lived gRPC connections [medium.com].
4. **Worker Processes (Your Compute):** Sits in your network [mikhail.io]. They continuously poll the Matching Service for tasks, execute the workflow or activity code, and return state changes back to the Frontend [mikhail.io].

------

## 5. Coding & Deployment Model

The following Python example shows how to declare an agent execution flow. Note how standard programming constructs (`try/except`, sequential assignments) are automatically made durable [temporal.io].

### Python Code Example

#### 1. Define the Activities (Non-Deterministic Tools & LLM calls)

```python
from temporalio import activity
import openai

@activity.defn
async def call_llm_agent(prompt: str) -> str:
    # Non-deterministic external calls live here
    response = openai.ChatCompletion.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content

@activity.defn
async def execute_database_tool(query: str) -> str:
    # Tool execution
    return f"Executed query: {query}"
```

#### 2. Define the Workflow (Deterministic Orchestration)

```python
from datetime import timedelta
from temporalio import workflow

# Import our activities
with workflow.unsafe.imports_passed_through():
    from activities import call_llm_agent, execute_database_tool

@workflow.defn
class DurableAgentWorkflow:
    @workflow.run
    async def run(self, initial_goal: str) -> str:
        current_state = initial_goal
        
        # Linear orchestration made completely durable
        for step in range(5): # Limit loop iterations to prevent history bloat
            # Call the LLM (Activity) with automatic retry policy
            agent_plan = await workflow.execute_activity(
                call_llm_agent,
                current_state,
                start_to_close_timeout=timedelta(minutes=2),
            )
            
            if "SUCCESS" in agent_plan:
                return agent_plan
                
            # Call the database tool (Activity)
            current_state = await workflow.execute_activity(
                execute_database_tool,
                agent_plan,
                start_to_close_timeout=timedelta(seconds=30),
            )
            
        # Reset event history for long-running workflows to avoid history bloat
        await workflow.continue_as_new(current_state)
```

### High-Level Deployment & Versioning Model

Deploying and updating Temporal applications avoids standard microservice deployment pitfalls (like downtime or mid-flight state corruption) via **Worker Versioning** [temporal.io].

```
   ┌────────────────────────────────────────────────────────┐
   │                  1. DEPLOY BUILD V1                    │
   │  - Workers register Build ID "v1.0"                    │
   │  - Workflow starts on "v1.0" worker                     │
   └────────────────────────────────────────────────────────┘
                               │
                               ▼
   ┌────────────────────────────────────────────────────────┐
   │                  2. UPGRADE CODE TO V2                 │
   │  - Modify prompt or add an orchestration step          │
   │  - Deploy Build ID "v2.0" parallel worker pool         │
   └────────────────────────────────────────────────────────┘
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
┌───────────────────────┐             ┌───────────────────────┐
│  IN-FLIGHT WORKFLOWS  │             │    NEW WORKFLOWS      │
│ - Retain "v1.0" Pin   │             │ - Route to "v2.0"     │
│ - Replayed on v1.0    │             │ - Evaluated on v2.0   │
└───────────────────────┘             └───────────────────────┘
```

1. **Build Tagging:** When deploying Worker containers, you register them under a specific logical Group Name and assign a unique **Build ID** (typically your Git commit hash or SemVer) [temporal.io].
2. Pinned Version Execution:
   - By default, when a workflow starts on Worker `v1.0`, it is **pinned** to that build [temporal.io].
   - When you release worker `v2.0` (which has updated prompt formats or tool mappings), the Temporal engine routes any continuing step or replay event for existing workflows *only* to the remaining `v1.0` workers [temporal.io].
3. **Draining:** Once all older workflows have completed naturally on your `v1.0` workers, those worker instances report "Drained" [temporal.io]. You can then safely decommission the physical servers/pods running the `v1.0` container, without ever having written inline compatibility code or causing execution downtime [temporal.io].

---

## Appendix: LangChain + LangGraph + Temporal Integration

This code blueprint demonstrates how to wrap a cyclical LangGraph agent inside a Temporal Workflow using the native `temporalio[langgraph]` integration [temporal.io]. 

The **LLM node** is declared as a **Temporal Activity**, ensuring that API timeouts, rate-limits, and network errors are handled with robust infra-level retries without restarting or losing the state of the parent graph [workos.com, temporal.io].

```python
from datetime import timedelta
from typing import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, END
from temporalio_langgraph import task, entrypoint, interrupt

# ==========================================
# 1. Cognitive Layer (LangChain)
# ==========================================
# Declare model and tools
model = ChatOpenAI(model="gpt-4o", temperature=0)

class AgentState(TypedDict):
    input: str
    output: str
    needs_approval: bool

# ==========================================
# 2. Behavioral Layer (LangGraph with Temporal)
# ==========================================
# "execute_in='activity'" wraps this node inside a durable Temporal Activity.
# If OpenAI rate-limits us, Temporal handles retries natively.
@task(execute_in="activity", start_to_close_timeout=timedelta(minutes=2))
def evaluate_goal(state: AgentState) -> AgentState:
    prompt = f"Develop a plan for: {state['input']}. If dangerous, say 'NEEDS_APPROVAL'."
    response = model.invoke(prompt)
    
    needs_approval = "NEEDS_APPROVAL" in response.content
    return {"output": response.content, "needs_approval": needs_approval}

# Node executes in the "workflow" context (deterministic routing).
@task(execute_in="workflow")
def request_human_approval(state: AgentState) -> AgentState:
    # Under Temporal, calling LangGraph's interrupt() automatically halts execution
    # and waits for an external gRPC "Signal" event. 
    # The worker dehydrates to 0 compute resources while waiting.
    decision = interrupt("Hold for human approval of safety risk.")
    return {"needs_approval": False, "output": f"Approved by human. Action: {decision}"}

# ==========================================
# 3. Compile the State Graph
# ==========================================
def route_approval(state: AgentState):
    return "human_approval" if state["needs_approval"] else END

# Define the cyclical agentic routing topology
workflow_builder = StateGraph(AgentState)
workflow_builder.add_node("evaluator", evaluate_goal)
workflow_builder.add_node("human_approval", request_human_approval)

workflow_builder.set_entry_point("evaluator")
workflow_builder.add_conditional_edges("evaluator", route_approval)
workflow_builder.add_edge("human_approval", END)

# ==========================================
# 4. Durable Orchestration Layer (Temporal Entrypoint)
# ==========================================
# The compiled LangGraph is decorated as a Temporal Workflow entrypoint
@entrypoint()
class DurableAgentGraph(workflow_builder.compile()):
    pass
```

### High-Level Deployment & Operation Steps

1.  **Package and Deploy the Worker:** Package this Python file inside a container and run it on your compute cluster (Kubernetes, AWS ECS, etc.) [mikhail.io]. Configure your Worker to poll a designated Task Queue (e.g., `agent-task-queue`) on your Temporal Cluster [mikhail.io, temporal.io].
2.  **Start Execution:** Trigger the workflow via the Temporal Client SDK:
    
    ```python
    await client.start_workflow(
        DurableAgentGraph.run,
        {"input": "Query database and initiate server migration."},
        id="migration-agent-001",
        task_queue="agent-task-queue"
    )
    ```
3.  **Survive Crash:** If your worker pod loses power mid-execution, Temporal Cloud (or your self-hosted cluster) detects the offline worker, assigns the execution history to an active worker, replays the event history, and resumes the exact node within LangGraph without repeating past LLM actions [workos.com, temporal.io].
4.  **Signal from Slack/Web-UI:** When the workflow hits the `interrupt()` block, resume it by sending a Signal to the workflow ID:
    ```python
    await client.get_workflow_handle("migration-agent-001").signal(
        "human_approval", "Proceed with caution."
    )
    ```
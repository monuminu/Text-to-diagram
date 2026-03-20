# Amazon Bedrock AgentCore: A Comprehensive Whitepaper

**Perspective:** GenAI Architect | **Date:** March 2026 | **Version:** 1.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Component Deep Dives](#component-deep-dives)
   - [3.1 Runtime](#31-runtime)
   - [3.2 Gateway](#32-gateway)
   - [3.3 Policy](#33-policy)
   - [3.4 Memory](#34-memory)
   - [3.5 Identity](#35-identity)
   - [3.6 Evaluations](#36-evaluations)
   - [3.7 Observability](#37-observability)
   - [3.8 Code Interpreter](#38-code-interpreter)
   - [3.9 Browser Tool](#39-browser-tool)
4. [Protocol Support: MCP and A2A](#protocol-support-mcp-and-a2a)
5. [Ecosystem Comparison](#ecosystem-comparison)
6. [Pricing and Availability](#pricing-and-availability)
7. [Recommendations and Best Practices](#recommendations-and-best-practices)
8. [Conclusion](#conclusion)

---

## Executive Summary

Amazon Bedrock AgentCore is a fully managed infrastructure service from AWS designed to deploy, secure, manage, and monitor AI agents at production scale. Announced at AWS re:Invent 2025 in preview and reaching **General Availability (GA) in July 2025**, AgentCore addresses the critical gap between prototype AI agents and production-ready deployments.

The core thesis of AgentCore is **framework agnosticism**. Unlike opinionated agent frameworks that dictate how agents are built, AgentCore provides the underlying infrastructure layer -- the "operating system" for agents. Whether agents are built with LangGraph, CrewAI, Strands Agents SDK, AutoGen, or custom Python/Java code, AgentCore provides a consistent set of managed services for runtime execution, tool access, security, memory, identity, evaluation, and observability.

### Strategic Positioning

AgentCore sits at a unique intersection in the AWS AI stack:

- **Below** the fully managed Bedrock Agents (which provides an opinionated, declarative agent building experience)
- **Above** raw infrastructure like EC2, Lambda, and ECS
- **Alongside** the open-source Strands Agents SDK (which provides the orchestration logic)

This positioning allows AgentCore to serve organizations that need production-grade infrastructure but want to retain control over their agent's orchestration logic, model selection, and framework choice.

### Key Value Propositions

| Capability | Without AgentCore | With AgentCore |
|---|---|---|
| Deployment | Custom container orchestration | Serverless, auto-scaling microVMs |
| Tool Access | Build and maintain connectors | 1-click integrations via Gateway |
| Security | DIY policy enforcement | Cedar-based deterministic guardrails |
| Memory | Custom database integrations | Managed short-term and long-term memory |
| Auth | Complex OAuth implementation | Managed identity with token vault |
| Monitoring | Custom instrumentation | Native OpenTelemetry + CloudWatch |
| Evaluation | Ad-hoc testing | 13 built-in evaluators + custom |

---

## Architecture Overview

AgentCore is composed of **10 modular, independently usable components** that together form a comprehensive agent infrastructure platform. Each component is accessible via AWS APIs and SDKs, and can be adopted incrementally.

```
                          +---------------------------+
                          |      Agent Application     |
                          | (Any Framework / Custom)   |
                          +---------------------------+
                                      |
                    +-----------------+-----------------+
                    |                                   |
              +-----v------+                    +-------v--------+
              |  Runtime    |                    |    Gateway      |
              | (Execution) |                    | (Tool Access)   |
              +-----+------+                    +-------+--------+
                    |                                   |
        +-----------+-----------+           +-----------+----------+
        |           |           |           |           |          |
   +----v---+ +----v----+ +----v---+  +----v----+ +----v---+ +----v--------+
   | Memory | | Policy  | |Identity|  |  Code   | |Browser | | Observ-     |
   |        | |         | |        |  |Interpret| | Tool   | | ability     |
   +--------+ +---------+ +--------+  +---------+ +--------+ +-------------+
                    |
              +-----v------+
              | Evaluations |
              +-------------+
```

### Design Principles

1. **Modularity**: Each component is independently deployable. Use only what you need.
2. **Framework Agnosticism**: No lock-in to a specific agent framework or LLM provider.
3. **Security by Default**: Identity, policy, and isolation are first-class concerns, not afterthoughts.
4. **Open Standards**: Built on OpenTelemetry, MCP, A2A, OAuth 2.0, and Cedar.
5. **Serverless-First**: Consumption-based pricing with zero idle cost and automatic scaling.

### Component Interaction Model

The components interact through well-defined interfaces:

- **Runtime** hosts and executes agent code, providing the compute substrate.
- **Gateway** provides tool discovery and invocation, acting as a universal tool router.
- **Policy** intercepts agent actions and enforces guardrails at the infrastructure level.
- **Memory** persists context across sessions and conversations.
- **Identity** manages authentication for both inbound requests and outbound tool calls.
- **Evaluations** assesses agent quality in real-time and on-demand.
- **Observability** captures telemetry across all components.
- **Code Interpreter** and **Browser Tool** are managed tools available through Gateway.

---

## Component Deep Dives

### 3.1 Runtime

**Purpose:** Serverless execution environment for deploying AI agents as managed endpoints.

AgentCore Runtime eliminates the need to manage infrastructure for agent deployment. It provides a serverless compute platform specifically optimized for AI agent workloads, which have unique characteristics -- long-running sessions, bursty tool calls, and variable latency from LLM inference.

#### Execution Model

Runtime uses **Firecracker microVMs** for agent isolation. Each agent instance runs in its own lightweight virtual machine, providing:

- **Strong isolation**: Agents from different customers (or different agents within the same account) cannot interfere with each other.
- **Fast cold starts**: Firecracker microVMs boot in ~125ms, enabling rapid scaling.
- **Resource guarantees**: Each microVM gets dedicated CPU and memory allocations.

#### Deployment

Agents are deployed as OCI-compliant container images. The deployment workflow:

1. Package agent code and dependencies into a Docker container.
2. Push to Amazon ECR.
3. Create an AgentCore Runtime endpoint specifying the container image.
4. Runtime handles provisioning, scaling, and lifecycle management.

```python
# Example: Creating a Runtime endpoint
import boto3

client = boto3.client('bedrock-agentcore')

response = client.create_agent_runtime(
    agentRuntimeName='my-agent',
    agentRuntimeArtifact={
        'containerImage': {
            'uri': '123456789012.dkr.ecr.us-east-1.amazonaws.com/my-agent:latest'
        }
    },
    networkConfiguration={
        'networkMode': 'PUBLIC'
    },
    protocolConfiguration={
        'serverProtocol': 'HTTP'  # Also supports MCP and A2A
    }
)
```

#### Protocol Support

Runtime endpoints natively support three protocols:

| Protocol | Use Case | Details |
|---|---|---|
| **HTTP** | Standard REST APIs | Request/response and streaming |
| **MCP** | Model Context Protocol | Standardized tool/resource serving |
| **A2A** | Agent-to-Agent | Google's multi-agent communication protocol |

This multi-protocol support means a single agent can serve as a REST endpoint for applications, an MCP server for other agents, or an A2A participant in multi-agent workflows -- without code changes.

#### Auto-Scaling

Runtime provides automatic scaling based on request load:

- **Scale to zero**: No charges when no requests are active (true serverless).
- **Burst scaling**: Rapid scale-up for sudden traffic spikes.
- **Configurable limits**: Set minimum and maximum instance counts.
- **Session affinity**: Long-running agent sessions maintain state on the same instance.

#### Networking

Two network modes are available:

- **PUBLIC**: Agent endpoint is accessible from the internet with IAM authentication.
- **VPC**: Agent runs within a customer VPC, enabling access to private resources (databases, internal APIs) while maintaining isolation.

---

### 3.2 Gateway

**Purpose:** Universal tool access layer that provides agents with discoverable, composable tools via a single API.

The Gateway is one of AgentCore's most differentiated components. It solves the **tool integration problem** -- instead of each agent implementing custom connectors for every service, Gateway provides a managed catalog of tools that agents can discover and invoke dynamically.

#### Tool Composition and Discovery

Gateway operates as a **tool router**. Agents don't need to know in advance which tools they'll need. Instead, they can:

1. **Describe their intent** to Gateway.
2. Gateway performs **semantic tool selection** to find relevant tools.
3. The agent invokes selected tools through a uniform API.

This is powered by an embedded retrieval system that matches natural language queries against tool descriptions, parameters, and capabilities.

#### MCP Translation Layer

A key architectural insight: Gateway normalizes all tools to the **Model Context Protocol (MCP)** interface. Regardless of the underlying tool's native API (REST, GraphQL, gRPC, SDK), Gateway presents a uniform MCP-compatible interface.

This means:
- Tools written as MCP servers are natively supported.
- Legacy REST APIs are wrapped with MCP-compatible metadata.
- Agents interact with a single protocol regardless of tool diversity.

#### 1-Click Integrations

Gateway provides pre-built, managed integrations with popular services:

| Category | Integrations |
|---|---|
| **Productivity** | Slack, Microsoft Teams, Gmail, Google Calendar |
| **Development** | GitHub, GitLab, Jira, Confluence |
| **Data** | Snowflake, Salesforce, HubSpot, Zendesk |
| **AWS Services** | S3, DynamoDB, Lambda, CloudWatch |
| **Search** | Google Search, Brave Search |

These integrations are fully managed -- AWS handles API versioning, authentication, rate limiting, and error handling.

#### Custom Tool Registration

Organizations can register their own tools with Gateway:

```python
# Register a custom tool with Gateway
response = client.create_gateway_tool(
    gatewayId='my-gateway',
    toolName='internal-crm-lookup',
    toolDescription='Look up customer information in the internal CRM system',
    toolSchema={
        'inputSchema': {
            'type': 'object',
            'properties': {
                'customerId': {'type': 'string', 'description': 'Customer ID'}
            },
            'required': ['customerId']
        }
    },
    toolEndpoint={
        'uri': 'https://internal-crm.company.com/api/lookup',
        'authType': 'IAM'
    }
)
```

#### Semantic Tool Selection

When an agent has access to hundreds of tools, including all of them in every LLM prompt is impractical (context window limits, cost, and confusion). Gateway solves this with **semantic tool selection**:

1. Agent sends a natural language query describing what it needs.
2. Gateway uses embeddings to find the top-K most relevant tools.
3. Only relevant tool schemas are returned to the agent.

This reduces token usage, improves agent accuracy, and enables scaling to large tool catalogs without degradation.

---

### 3.3 Policy

**Purpose:** Deterministic guardrails that enforce security and compliance policies on agent actions.

Policy is AgentCore's answer to a fundamental production concern: **How do you ensure an autonomous agent doesn't perform unauthorized or dangerous actions?** Unlike probabilistic LLM-based guardrails, AgentCore Policy provides **deterministic enforcement** -- policies always execute the same way regardless of LLM behavior.

#### Cedar Policy Language

AgentCore Policy is built on **Cedar**, an open-source policy language developed by AWS (also used in Amazon Verified Permissions). Cedar provides:

- **Formal verification**: Policies can be mathematically proven correct.
- **Fast evaluation**: Sub-millisecond policy decisions.
- **Expressive syntax**: Natural, readable policy definitions.

```cedar
// Allow agents to read from S3, but only in the approved bucket
permit(
    principal == Agent::"customer-service-agent",
    action == Action::"s3:GetObject",
    resource
) when {
    resource.bucket == "approved-data-bucket"
};

// Deny all agents from deleting production resources
forbid(
    principal,
    action == Action::"ec2:TerminateInstances",
    resource
) when {
    resource.environment == "production"
};
```

#### Natural Language Policy Authoring

For teams without Cedar expertise, AgentCore provides a **natural language to Cedar compiler**. Users can describe policies in plain English, and AgentCore generates the corresponding Cedar policy:

| Natural Language | Generated Cedar |
|---|---|
| "The agent can only send emails to @company.com addresses" | `forbid(...) unless { resource.recipientDomain == "company.com" }` |
| "Limit API calls to 100 per hour" | Rate-limiting policy with counter conditions |
| "Never share PII in responses" | Content inspection policy with PII detection |

#### Enforcement Modes

AgentCore Policy supports two operational modes:

- **Enforce Mode**: Policy violations are blocked. The agent receives an access-denied response and must find an alternative approach.
- **Log Mode**: Policy violations are logged but allowed to proceed. This is useful for testing new policies before enforcement, observing agent behavior, and gradual rollout of restrictions.

#### Policy Architecture

Policies are evaluated **at the infrastructure level**, not within the agent's code. This means:

1. Even if agent code has bugs or is compromised, policies still enforce.
2. Policies apply consistently across all agents in the account.
3. No performance overhead on the agent -- policy evaluation happens in the control plane.

#### Scope and Granularity

Policies can be scoped at multiple levels:

- **Account-level**: Apply to all agents in an AWS account.
- **Agent-level**: Apply to a specific agent runtime.
- **Session-level**: Apply to a specific user session or conversation.
- **Action-level**: Apply to specific tool calls, API actions, or resource types.

---

### 3.4 Memory

**Purpose:** Managed persistence layer that gives agents the ability to remember context across sessions, conversations, and time.

Memory is what transforms a stateless LLM into a contextually aware agent. AgentCore Memory provides a fully managed service for storing, retrieving, and managing agent memory at scale.

#### Memory Types

AgentCore distinguishes between two fundamental memory categories:

**Short-Term Memory (Working Memory)**
- Stores the current session's context: conversation history, intermediate results, scratchpad data.
- Automatically managed within a session.
- TTL-based expiration (configurable).
- Equivalent to a human's working memory during a task.

**Long-Term Memory (Persistent Memory)**
- Persists across sessions and conversations.
- Stores learned facts, user preferences, historical interactions.
- Supports multiple retrieval strategies.
- Equivalent to a human's episodic and semantic memory.

#### Memory Strategies

AgentCore supports multiple memory consolidation strategies that determine how raw interactions are processed into retrievable memories:

| Strategy | Description | Best For |
|---|---|---|
| **Semantic** | Extracts key facts and entities from conversations | Knowledge-heavy agents (research, Q&A) |
| **Summary** | Generates concise summaries of interactions | Customer service, long conversations |
| **Episodic** | Stores interaction sequences as episodes | Task-oriented agents needing step recall |
| **User Preference** | Extracts and maintains user-specific preferences | Personalization-heavy agents |

#### Reflections

A powerful feature of AgentCore Memory is **reflections** -- the ability for agents to synthesize higher-order insights from accumulated memories. Reflections are periodically generated meta-memories that identify patterns across individual memories:

```
Individual Memories:
  - "User prefers dark mode in all applications"
  - "User asked for compact view three times"
  - "User disabled email notifications"

Reflection:
  "User strongly prefers minimal, low-stimulus interfaces. Proactively suggest
   compact/dark modes and limited notifications for new tools."
```

Reflections enable agents to develop increasingly nuanced understanding of users and contexts over time.

#### Namespaces

Memories are organized into **namespaces** that provide logical separation:

- **User namespace**: Memories scoped to a specific end-user.
- **Agent namespace**: Memories shared across all sessions of an agent.
- **Session namespace**: Memories scoped to a specific conversation.
- **Global namespace**: Cross-agent shared memories (e.g., organizational knowledge).

```python
# Store a long-term memory
client.create_memory(
    memoryId='agent-memory-store',
    namespace='user/user-123',
    content={
        'text': 'User prefers Python over Java for code examples',
        'type': 'USER_PREFERENCE'
    },
    strategy='SEMANTIC'
)

# Retrieve relevant memories
memories = client.retrieve_memory(
    memoryId='agent-memory-store',
    namespace='user/user-123',
    query='What programming language does the user prefer?',
    maxResults=5
)
```

#### Memory Lifecycle

AgentCore handles the full memory lifecycle:

1. **Ingestion**: Raw conversation data is ingested.
2. **Processing**: Selected strategy extracts structured memories.
3. **Storage**: Memories are stored with vector embeddings for retrieval.
4. **Retrieval**: Semantic search finds relevant memories for current context.
5. **Consolidation**: Periodic reflections synthesize higher-order patterns.
6. **Expiration**: TTL-based cleanup prevents unbounded growth.

---

### 3.5 Identity

**Purpose:** Managed authentication and authorization for both inbound requests to agents and outbound calls from agents to external services.

Identity is a critical production concern that most agent prototypes ignore. In production, agents need to authenticate users making requests, authenticate themselves when calling external APIs, manage OAuth flows, and securely store and refresh tokens. AgentCore Identity handles all of this as a managed service.

#### Inbound Authentication

Inbound identity verifies **who is calling the agent**:

- **IAM Authentication**: AWS Signature V4 for service-to-service calls.
- **OAuth 2.0 / OIDC**: For end-user authentication via identity providers (IdPs).
- **API Keys**: Simple key-based authentication for internal services.
- **Custom Authenticators**: Pluggable authentication for proprietary identity systems.

#### Outbound Authentication

Outbound identity manages **how the agent authenticates with external services**. This is where AgentCore provides significant value:

**The Problem**: An agent needs to call GitHub, Salesforce, and Slack APIs on behalf of a user. Each service has different OAuth flows, token formats, refresh mechanisms, and scopes.

**The Solution**: AgentCore Identity abstracts all of this:

1. **Token Vault**: Securely stores OAuth tokens, API keys, and credentials (encrypted at rest with AWS KMS).
2. **Automatic Token Refresh**: Handles OAuth refresh token flows automatically.
3. **Consent Management**: Manages user consent for third-party service access.
4. **Credential Rotation**: Supports automatic credential rotation policies.

#### Built-In Identity Providers

AgentCore provides pre-configured integrations with common identity providers:

| Provider | Auth Flow | Use Case |
|---|---|---|
| Google | OAuth 2.0 + OIDC | Gmail, Calendar, Drive access |
| Microsoft | OAuth 2.0 + OIDC | Outlook, Teams, SharePoint access |
| GitHub | OAuth 2.0 | Repository, issue, PR access |
| Salesforce | OAuth 2.0 | CRM data access |
| Slack | OAuth 2.0 | Messaging, channel access |
| Custom OIDC | OAuth 2.0 + OIDC | Enterprise SSO systems |

#### OAuth Flow Management

AgentCore manages the complete OAuth lifecycle:

```
User Request --> Agent needs GitHub data
                    |
                    v
         AgentCore Identity checks Token Vault
                    |
            +-------+-------+
            |               |
      Token exists    No token found
      & valid              |
            |               v
            |         Initiate OAuth flow
            |         (redirect user to GitHub)
            |               |
            |               v
            |         User authorizes
            |               |
            |               v
            |         Store tokens in vault
            |               |
            v               v
         Use token to call GitHub API
                    |
                    v
         Return results to agent
```

#### Security Architecture

- All tokens are encrypted at rest using AWS KMS customer-managed keys (CMKs).
- Token access is governed by IAM policies -- agents can only access tokens they're authorized to use.
- Audit logging via CloudTrail captures all token access events.
- Tokens are never exposed in logs, traces, or error messages.

---

### 3.6 Evaluations

**Purpose:** Continuous quality assessment for AI agents through built-in and custom evaluators in both online (real-time) and on-demand modes.

Evaluation is one of the hardest problems in production AI systems. Unlike traditional software with deterministic outputs, agent behavior is probabilistic and context-dependent. AgentCore Evaluations provides a structured approach to measuring and monitoring agent quality.

#### 13 Built-In Evaluators

AgentCore ships with 13 pre-built evaluators spanning multiple quality dimensions:

| Evaluator | Category | What It Measures |
|---|---|---|
| **Faithfulness** | Accuracy | Are responses grounded in provided context? |
| **Answer Relevance** | Accuracy | Does the response address the user's question? |
| **Context Relevance** | Retrieval | Is the retrieved context relevant to the query? |
| **Context Utilization** | Retrieval | Does the response effectively use available context? |
| **Hallucination** | Accuracy | Does the response contain fabricated information? |
| **Harmfulness** | Safety | Does the response contain harmful content? |
| **Maliciousness** | Safety | Does the response exhibit malicious intent? |
| **Toxicity** | Safety | Does the response contain toxic language? |
| **PII Detection** | Compliance | Does the response expose personally identifiable information? |
| **SQL Correctness** | Tool Use | Are generated SQL queries syntactically correct? |
| **Tool Selection** | Tool Use | Did the agent select the appropriate tool? |
| **Tool Parameter** | Tool Use | Were tool parameters populated correctly? |
| **Custom** | Flexible | User-defined evaluation criteria |

#### LLM-as-Judge

The built-in evaluators use an **LLM-as-Judge** pattern, where a separate LLM evaluates the primary agent's outputs. Key design decisions:

- The judge LLM is a different model instance to avoid self-evaluation bias.
- Evaluators use structured rubrics with specific scoring criteria.
- Scores are normalized to a 0-1 scale for cross-evaluator comparison.
- Explanations are provided alongside scores for debuggability.

#### Online vs. On-Demand Evaluation

**Online (Real-Time) Evaluation**
- Evaluators run in the request path (asynchronously, non-blocking).
- Every agent response is evaluated.
- Results feed into CloudWatch metrics for dashboards and alerts.
- Use case: Continuous monitoring of production agents.

**On-Demand (Batch) Evaluation**
- Run evaluators against a dataset of test cases.
- Compare agent versions (A/B testing).
- Generate comprehensive evaluation reports.
- Use case: Pre-deployment validation, regression testing.

#### Custom Evaluators

Organizations can define custom evaluators for domain-specific quality criteria:

```python
# Define a custom evaluator
response = client.create_evaluator(
    evaluatorName='financial-accuracy',
    evaluatorType='LLM_AS_JUDGE',
    evaluationPrompt="""
    You are evaluating a financial advisory agent's response.

    Score the response on a 1-5 scale for:
    1. Numerical accuracy of financial calculations
    2. Appropriate use of disclaimers
    3. Regulatory compliance (SEC/FINRA guidelines)

    Provide a JSON response with scores and explanations.
    """,
    scoringRubric={
        '5': 'Fully accurate, compliant, with appropriate disclaimers',
        '1': 'Contains numerical errors or regulatory violations'
    }
)
```

#### Evaluation Pipeline

```
Agent Response
      |
      v
  +---+---+
  | Route  | --> Online evaluators (async, real-time)
  +---+---+     |
      |         +--> Faithfulness score --> CloudWatch metric
      |         +--> Harmfulness score --> CloudWatch metric
      |         +--> Custom score --> CloudWatch metric
      |
      v
  Response delivered to user
  (evaluation runs in parallel, does not block response)
```

---

### 3.7 Observability

**Purpose:** End-to-end visibility into agent behavior through distributed tracing, metrics, and logging built on OpenTelemetry standards.

Observability is non-negotiable for production AI agents. When an agent makes unexpected decisions, takes too long, or fails silently, operators need the tools to diagnose and remediate. AgentCore Observability provides this through deep integration with OpenTelemetry and AWS CloudWatch.

#### OpenTelemetry Foundation

AgentCore Observability is built on the **OpenTelemetry (OTel)** standard, ensuring vendor-neutral telemetry collection. This means:

- Telemetry data can be exported to any OTel-compatible backend (Datadog, Grafana, Splunk, etc.).
- Standard OTel SDKs and instrumentation work out of the box.
- No proprietary lock-in for observability data.

#### ADOT SDK Integration

AgentCore provides the **AWS Distro for OpenTelemetry (ADOT) SDK** with agent-specific instrumentation:

- **Auto-instrumentation**: Automatically captures LLM calls, tool invocations, memory operations, and policy decisions without code changes.
- **Semantic conventions**: Agent-specific span attributes (model ID, token counts, tool names) follow standardized naming.
- **Context propagation**: Distributed traces flow across agent-to-agent calls, tool invocations, and async operations.

#### Telemetry Hierarchy

AgentCore organizes telemetry into three levels:

```
Session (top-level)
  |
  +-- Trace (single request/response cycle)
        |
        +-- Span: LLM Inference
        |     +-- Attributes: model_id, input_tokens, output_tokens, latency
        |
        +-- Span: Tool Invocation
        |     +-- Attributes: tool_name, gateway_id, status, duration
        |
        +-- Span: Memory Retrieval
        |     +-- Attributes: namespace, results_count, relevance_scores
        |
        +-- Span: Policy Evaluation
              +-- Attributes: policy_id, decision (allow/deny), evaluation_time
```

#### CloudWatch Integration

AgentCore automatically publishes structured telemetry to CloudWatch:

- **Dashboards**: Pre-built dashboards for agent health, performance, and quality.
- **Metrics**: Latency percentiles, error rates, token usage, tool call frequency.
- **Logs**: Structured JSON logs with correlation IDs for tracing.
- **Alarms**: Configurable alerts for anomalies (e.g., hallucination rate spike, latency degradation).

#### Key Metrics

| Metric | Description | Alert Threshold Example |
|---|---|---|
| `AgentLatency_p99` | 99th percentile end-to-end latency | > 30 seconds |
| `ToolCallErrorRate` | Percentage of failed tool invocations | > 5% |
| `TokenUsagePerSession` | Average tokens consumed per session | > 100,000 |
| `PolicyDenyRate` | Percentage of actions blocked by policy | > 10% (may indicate misconfiguration) |
| `HallucinationScore_avg` | Average hallucination evaluator score | < 0.7 |
| `MemoryRetrievalLatency` | Time to retrieve relevant memories | > 500ms |

#### Session Replay

A distinctive feature of AgentCore Observability is **session replay** -- the ability to reconstruct an agent's complete decision-making process for a given session:

1. Every LLM prompt and response is captured.
2. Every tool call with inputs and outputs is logged.
3. Every policy decision is recorded.
4. Every memory retrieval and its relevance scores are traced.

This enables post-hoc debugging of agent behavior, compliance auditing, and root cause analysis for failures.

---

### 3.8 Code Interpreter

**Purpose:** Managed sandbox environment for agents to execute code dynamically, supporting multi-language execution with S3 integration.

Many agent use cases require dynamic code execution -- data analysis, visualization, mathematical computation, file transformation, and more. AgentCore Code Interpreter provides a secure, managed sandbox for this purpose.

#### Sandbox Architecture

Each Code Interpreter session runs in an **isolated sandbox environment**:

- **Container-based isolation**: Each execution environment is a fresh container.
- **Resource limits**: CPU, memory, and execution time are bounded.
- **No persistent state**: Sandboxes are ephemeral (stateless between invocations unless explicitly configured).
- **File system isolation**: Agents can only access files explicitly provided.

#### Multi-Language Support

Code Interpreter supports multiple programming languages:

| Language | Use Cases |
|---|---|
| **Python** | Data analysis (pandas, numpy), visualization (matplotlib), ML inference |
| **JavaScript/Node.js** | Web scraping, JSON transformation, API integration |
| **Shell/Bash** | File manipulation, system operations, piping |

Python is the primary supported language with the richest set of pre-installed packages.

#### S3 Integration

Code Interpreter integrates directly with Amazon S3 for file I/O:

- **Input**: Agents can reference S3 objects as input files for code execution.
- **Output**: Generated files (charts, reports, transformed data) are automatically uploaded to S3.
- **Large file support**: Process files larger than what fits in an LLM context window.

```python
# Agent instructs Code Interpreter to analyze data
code_interpreter_input = {
    'code': '''
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('/input/sales_data.csv')
monthly = df.groupby('month')['revenue'].sum()
monthly.plot(kind='bar', title='Monthly Revenue')
plt.savefig('/output/revenue_chart.png')
print(f"Total revenue: ${monthly.sum():,.2f}")
    ''',
    'inputFiles': [
        {'s3Uri': 's3://my-bucket/sales_data.csv', 'mountPath': '/input/sales_data.csv'}
    ],
    'outputS3Uri': 's3://my-bucket/outputs/'
}
```

#### Network Modes

Code Interpreter supports two network configurations:

- **No Network (Default)**: The sandbox has no internet access. This is the most secure mode, preventing data exfiltration and ensuring reproducibility.
- **Network Enabled**: The sandbox can access the internet. Required for use cases like fetching live data, calling APIs, or installing packages. Controlled by Policy guardrails.

#### Pre-Installed Packages

The Python sandbox comes with a curated set of pre-installed packages:

- **Data**: pandas, numpy, scipy, scikit-learn
- **Visualization**: matplotlib, seaborn, plotly
- **File Processing**: openpyxl, python-docx, PyPDF2, Pillow
- **Utilities**: requests, beautifulsoup4, json, csv

---

### 3.9 Browser Tool

**Purpose:** Managed headless browser automation that enables agents to interact with web applications, fill forms, extract data, and navigate websites.

The Browser Tool brings web interaction capabilities to AI agents through a managed headless browser service. This is essential for agents that need to interact with web applications that lack APIs, perform web research, or automate web-based workflows.

#### Architecture

Browser Tool is built on **Playwright**, the open-source browser automation framework by Microsoft. AgentCore provides a managed, cloud-hosted version:

- **Headless Chrome/Chromium**: Full browser rendering without a GUI.
- **Managed infrastructure**: AWS handles browser instance provisioning, scaling, and lifecycle.
- **Session management**: Each agent session gets an isolated browser instance.

#### Capabilities

| Capability | Description |
|---|---|
| **Navigation** | Navigate to URLs, follow links, handle redirects |
| **Interaction** | Click buttons, fill forms, select dropdowns, scroll |
| **Extraction** | Extract text, structured data, screenshots |
| **Waiting** | Wait for elements, network requests, or conditions |
| **Multi-tab** | Open and manage multiple browser tabs |
| **File handling** | Upload and download files through the browser |

#### Web Bot Authentication (Web Bot Auth)

A standout feature is **Web Bot Auth** -- the ability for agents to authenticate to websites:

- Integrates with AgentCore Identity's Token Vault.
- Supports OAuth-based web login flows.
- Handles session cookies and authentication state.
- Enables agents to access authenticated web content on behalf of users.

#### CAPTCHA Reduction

AgentCore Browser Tool includes built-in CAPTCHA reduction mechanisms:

- Legitimate browser fingerprinting to reduce CAPTCHA triggering.
- Standard browser headers and behavior patterns.
- Rate limiting to avoid triggering anti-bot protections.

**Note**: This is designed for **legitimate automation** (accessing services the user has authorized), not for bypassing security controls on unauthorized services.

#### Integration with Gateway

Browser Tool is registered as a managed tool in Gateway. Agents discover and invoke it through the standard Gateway interface:

```python
# Agent uses Browser Tool through Gateway
browser_result = gateway.invoke_tool(
    toolName='aws:browser',
    input={
        'action': 'navigate',
        'url': 'https://example-internal-dashboard.com/reports',
        'extractSelectors': {
            'title': 'h1.report-title',
            'data': 'table.report-data'
        }
    }
)
```

#### Observability

All browser actions are fully traced:
- Each navigation, click, and extraction is captured as a span.
- Screenshots can be captured at each step for debugging.
- Network requests made by the browser are logged.
- Page load times and interaction latencies are metriced.

---

## Protocol Support: MCP and A2A

AgentCore's protocol support is a strategic differentiator that positions it for the emerging multi-agent ecosystem.

### Model Context Protocol (MCP)

MCP, developed by Anthropic and adopted as an open standard, defines a standardized interface for tools and resources that LLMs can interact with. AgentCore's MCP support means:

**As MCP Client (Agent calling tools)**:
- Agents can invoke any MCP-compatible server as a tool.
- Gateway translates non-MCP tools into MCP interfaces.
- Standard MCP discovery and schema negotiation is supported.

**As MCP Server (Agent serving capabilities)**:
- Runtime can expose agents as MCP servers.
- Other agents or applications can discover and invoke the agent's capabilities.
- Tool schemas are automatically generated from agent capabilities.

### Agent-to-Agent Protocol (A2A)

A2A, developed by Google, enables structured communication between agents from different frameworks and vendors. AgentCore's A2A support includes:

- **Agent Cards**: Agents publish discoverable capability descriptions.
- **Task Management**: Structured task lifecycle (create, update, complete, cancel).
- **Streaming**: Real-time updates during long-running agent tasks.
- **Push Notifications**: Asynchronous callbacks for task completion.

### Multi-Agent Architectures

With both MCP and A2A support, AgentCore enables sophisticated multi-agent architectures:

```
                    +------------------+
                    | Orchestrator     |
                    | Agent (A2A)      |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +-----v------+
     | Research    |  | Analysis   |  | Writing    |
     | Agent (MCP) |  | Agent (MCP)|  | Agent (A2A)|
     +--------+---+  +------+-----+  +-----+------+
              |              |              |
         +----v----+    +----v----+    +----v----+
         | Web     |    | Code    |    | Doc     |
         | Search  |    |Interpret|    | Store   |
         | (Tool)  |    | (Tool)  |    | (Tool)  |
         +---------+    +---------+    +---------+
```

---

## Ecosystem Comparison

Understanding where AgentCore fits relative to other AWS AI agent services is critical for architectural decisions.

### AgentCore vs. Bedrock Agents vs. Strands Agents SDK

| Dimension | Bedrock Agents | AgentCore | Strands Agents SDK |
|---|---|---|---|
| **Abstraction Level** | Highest (declarative) | Middle (infrastructure) | Lowest (code) |
| **Framework Lock-in** | Yes (Bedrock-native) | No (any framework) | Minimal (Python SDK) |
| **Deployment** | Fully managed | Managed infrastructure | Self-managed or AgentCore |
| **Customization** | Limited | High | Complete |
| **Orchestration** | Built-in (ReAct/CoT) | BYO | Built-in (extensible) |
| **Best For** | Rapid prototyping, simple agents | Production at scale | Custom orchestration logic |
| **Model Support** | Bedrock models only | Any model | Any model |
| **Pricing** | Per-step | Per-compute | Open source (free) |

### When to Use What

**Choose Bedrock Agents when:**
- Building simple, well-defined agent workflows
- Want fastest time-to-production
- Don't need custom orchestration
- Comfortable with Bedrock model ecosystem

**Choose AgentCore when:**
- Building complex, multi-framework agent systems
- Need production-grade infrastructure (security, observability, memory)
- Want framework and model flexibility
- Deploying agents at enterprise scale
- Need multi-protocol support (MCP, A2A)

**Choose Strands Agents SDK when:**
- Want maximum control over agent logic
- Building research or experimental agents
- Need custom orchestration patterns
- Want open-source with no infrastructure lock-in
- Can deploy on AgentCore Runtime for production

### Complementary Usage

These services are complementary, not competitive:

```
Strands SDK (orchestration logic)
    +
AgentCore (production infrastructure)
    =
Production-ready custom agents

Bedrock Agents (simple, managed agents)
    +
AgentCore Gateway (tool access for other agents)
    =
Hybrid architecture
```

---

## Pricing and Availability

### Consumption-Based Pricing

AgentCore follows a **pay-per-use** model with no upfront commitments:

| Component | Pricing Dimension | Notes |
|---|---|---|
| **Runtime** | vCPU-seconds + memory-seconds | Scale to zero; no idle charges |
| **Gateway** | Per tool invocation | Includes managed connector maintenance |
| **Policy** | Per policy evaluation | Sub-cent per evaluation |
| **Memory** | Per memory operation + storage | Storage billed per GB-month |
| **Identity** | Per authentication event | Token vault storage included |
| **Evaluations** | Per evaluation run | LLM judge inference billed separately |
| **Observability** | Standard CloudWatch pricing | Trace storage and metric charges |
| **Code Interpreter** | Per execution-second | Compute + S3 storage for files |
| **Browser Tool** | Per browser session-minute | Includes managed Chromium infrastructure |

### Regional Availability

AgentCore reached GA in **9 AWS regions** at launch:

| Region | Region Code |
|---|---|
| US East (N. Virginia) | us-east-1 |
| US East (Ohio) | us-east-2 |
| US West (Oregon) | us-west-2 |
| Europe (Frankfurt) | eu-central-1 |
| Europe (Ireland) | eu-west-1 |
| Europe (London) | eu-west-2 |
| Asia Pacific (Tokyo) | ap-northeast-1 |
| Asia Pacific (Singapore) | ap-southeast-1 |
| Asia Pacific (Sydney) | ap-southeast-2 |

---

## Recommendations and Best Practices

### Architecture Recommendations

1. **Start with Gateway + Identity**: Even before deploying agents on Runtime, integrate Gateway for tool access and Identity for auth. These provide immediate value regardless of where your agent runs.

2. **Adopt Policy Early**: Define guardrails before agents reach production. Use Log Mode to observe behavior, then switch to Enforce Mode. This is far easier than retrofitting policies onto a running system.

3. **Layer Memory Strategically**: Not every agent needs long-term memory. Start with short-term (session) memory and add long-term memory only when you have clear use cases (personalization, learning).

4. **Use Evaluations as CI/CD Gates**: Integrate on-demand evaluations into your deployment pipeline. Set minimum score thresholds for faithfulness and safety before promoting agent versions.

5. **Instrument from Day One**: Enable Observability from the first deployment. The cost is minimal, and the debugging value is immense. You cannot retroactively trace issues you didn't instrument.

### Security Best Practices

1. **Principle of Least Privilege**: Define Cedar policies that grant minimum required permissions. Start restrictive and widen based on observed needs (using Log Mode).

2. **Isolate by Workload**: Use separate Runtime endpoints for agents handling different sensitivity levels. Don't mix customer-facing agents with internal-only agents.

3. **Rotate Credentials**: Use Identity's automatic credential rotation. Never hardcode API keys in agent code.

4. **Audit Regularly**: Review CloudTrail logs for Identity token access patterns. Monitor Policy deny rates for unusual activity.

5. **Network Segmentation**: Use VPC mode for agents accessing sensitive internal resources. Apply security groups and NACLs as you would for any production workload.

### Operational Best Practices

1. **Set Up Alarms**: Configure CloudWatch alarms for key metrics -- latency spikes, error rate increases, evaluation score drops, and policy deny rate anomalies.

2. **Version Your Agents**: Use container image tags and Runtime endpoint versioning. Never deploy untagged images to production.

3. **Test with Evaluations**: Run the full evaluation suite against test datasets before every production deployment. Track scores over time to catch regressions.

4. **Monitor Token Usage**: LLM costs can escalate quickly. Use Observability to track token consumption per agent, per session, and per tool call. Set budget alerts.

5. **Design for Failure**: Agents will encounter tool failures, LLM errors, and policy denials. Design agent logic to handle these gracefully -- retry with backoff, fallback to alternative tools, or escalate to humans.

### Multi-Agent Design Patterns

1. **Supervisor Pattern**: A coordinator agent delegates to specialist agents. Use A2A protocol for structured task management between agents.

2. **Pipeline Pattern**: Sequential processing where each agent's output feeds the next. Use MCP for tool-like agent invocation.

3. **Consensus Pattern**: Multiple agents independently solve the same problem, and a judge agent evaluates responses. Use Evaluations to score outputs.

4. **Human-in-the-Loop**: Use Policy to require human approval for high-impact actions. Combine with Observability for audit trails.

---

## Conclusion

Amazon Bedrock AgentCore represents a maturation point in the AI agent ecosystem. By providing managed infrastructure for the hard problems of production deployment -- security, identity, memory, observability, and evaluation -- it allows engineering teams to focus on what differentiates their agents: the domain logic, user experience, and business value.

The key architectural insight of AgentCore is that **agent infrastructure should be decoupled from agent logic**. Just as Kubernetes decoupled application deployment from infrastructure management, AgentCore decouples agent deployment from the underlying concerns of execution, security, and monitoring.

For GenAI Architects evaluating AgentCore, the recommendation is clear: adopt the components that solve your immediate pain points (typically Runtime + Gateway + Identity), then expand to Memory, Policy, and Evaluations as your agent deployments mature. The modular design ensures you pay only for what you use and can adopt incrementally.

AgentCore is not a replacement for agent frameworks -- it's the production foundation they've been missing.

---

*This whitepaper reflects AgentCore capabilities as of its General Availability release. AWS services evolve rapidly; consult the [official AWS documentation](https://docs.aws.amazon.com/bedrock-agentcore) for the latest features and pricing.*

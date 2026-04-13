---
title: "Context-Aware Authorization for AI Agents: A Graph-Based Gateway"
date: 2026-04-10T16:58:19+02:00
showDate: false
draft: false
tags: ["blog","AI", "LLM", "A2A", "MCP", "proxy", "context graph", "graph db", "neo4j", "Go", "Golang", "back end"]
---

## Introduction

Standard identity models break when AI agents begin interacting autonomously. In a multi-agent workflow, **who** an agent is matters less than **why** they are calling. To solve this, we spent a week-long hackathon building a proof of concept for a **context-graph-based agent gateway**. This Policy Enforcement Point sits in front of every protected agent, evaluating the intent and lineage of every incoming request before execution.

By leveraging the A2A protocol, the gateway intercepts requests to validate a recursive delegation chain. Using a Graph DB, it ensures a verified relationship exists between all participants in a workflow before any code runs

### Why RBAC Fails in Agentic Workflows 
Traditional Role-Based Access Control (RBAC) is too static for dynamic agent chains. Consider this scenario:

- User `bob` is allowed to use `agent1` and `agent2`
- `agent1` is allowed to call, when required, `agent3`

If `bob` asks `agent1` to trigger `agent3` on his behalf, RBAC sees two valid hops and allows it. However, if Bob isn't supposed to access the data Agent 3 holds, this constitutes an unauthorized lateral move. In environments with thousands of interconnected agents, these "silent" permission escalations lead to the "AI deleted my DB" headlines we see today.

## Architecture Overview
Rather than a centralized bottleneck, we implemented a distributed "Sentinel" pattern. Each agent is shielded by its own dedicated Gateway instance.

![architecture overview](/posts/agent-gateway/architecture-overview.png)

The gateway is a stateless (utilizing in-memory caching) reverse proxy. It exposes a JSONRPC-over-HTTP interface and handles secure forwarding via the A2A protocol.

### Thech Stack in Short
- **Go + Gin**: High-performance gateway core.
- **Neo4j**: Context graph for relationship mapping.
- **A2A Protocol**: Standardized agent communication.
- **OAuth Identity Provider**: Identity and token verification.
- **REST API**: Communication with public enterprise APIs.

![detailed architecture](/posts/agent-gateway/architecture.png)

### Request Lifecycle

Every request processed by the gateway follows a deterministic pipeline:

1. Request received over HTTP
1. Caller token validated via IdP
1. Delegation chain extracted from token
1. Gateway authenticates as the protected agent
1. Graph queried to validate relationships
1. Policies evaluated
1. Decision produced (allow/deny)
1. Audit log generated
1. If allowed, request forwarded via A2A


### Token's Anatomy
The gateway relies on the `act` claim in JWTs to encode delegation.
The `act` claim is a **recursive object** that describes the delegation chain **in reverse order**.

![sample workflow](/posts/agent-gateway/workflow.png)
Given the above workflow, the token for a user (e.g. `alice`) that wants to call the chain `agent1->agent3->agent4` would be:
```json
{
    "sub": "alice",
    "act": {
        "sub": "agent4"
        "act": {
            "sub": "agent3"
            "act": {
                "sub": "agent1"
            }
        }
    }
}
```

### Authorization Exchange (AuthZEN)
Once the token is validated, the gateway uses the AuthZEN policy engine to ensure the user has permission to trigger the specific workflow. A sample policy might look like this:

```json
{
    "subject": {
        "type": "Person"
	},
	"actions": [
		"TRIGGERS"
	],
	"resource": {
		"type": "Workflow"
	},
    "condition": {
        "cypher": "MATCH (subject)-[:TRIGGERS]->(resource:Workflow)"
    }
}
```

The gateway confirms that at least one valid workflow in the graph contains the entire delegation chain. While partial chains are allowed (starting a workflow mid-way), partially invalid chains (i.e. where an agent doesn't belong to that specific context) are immediately rejected.

This policy ensures a subject of type `Person` has a relationship of type `TRIGGERS` that leads to a `Workflow` resource.

The model is schema-agnostic: terms like `workflow`, `agent`, or `TRIGGERS` are configurable and mapped via policy definitions.


### The whys and the trade-offs
Developing the gateway during a one-week hackathon forced us to be quick and strategic with our technical choices:

- **Go**: Selected for its concurrency primitives and memory efficiency, aligning with our internal company standards.
- **Caching**: To prevent database bottlenecks and excessive API calls, we implemented a 5-minute TTL in-memory cache for both workflows (retrieved from Neo4J) and agent clients (discovered through `/.well-known` HTTP calls). Future iterations will move toward a workflows eviction strategy driven by Neo4j Change Data Capture events.
- **A2A over MCP**: While we plan to support the Model Context Protocol (MCP), A2A was chosen for its maturity in agent-to-agent handshakes and rapid prototyping.
- **Distributed Gateways**: One gateway per agent ensures isolation and prevents a single point of failure, though it currently requires manual provisioning by the customer.


## Observability and Auditability

One of the primary goals of the gateway is to make agent systems **observable and auditable**.

Every authorization decision produces an audit log containing:

- The originating subject  
- The full delegation chain  
- The timestamp  
- The decision (allow or deny)  
- The reason for the decision  
- The service name to identify which gateway produced the log

Sensitive data (tokens and payloads) is never logged.
This ensures decisions are **traceable**, **explainable**, and **safe to store**, enabling post-mortem analysis and compliance auditing.

## Conclusion

The shift from human-to-machine to agent-to-agent interactions requires a fundamental rethink of the "trust but verify" principle. By moving security logic into a dedicated graph-based gateway, we **decouple identity from intent**. Our PoC demonstrates that we can enforce complex, multi-hop authorization without a centralized bottleneck or heavy latency penalties.

As AI agents move from simple chatbots to autonomous executors of business logic, the "Sentinel" pattern provides the necessary guardrails to ensure that an agent’s autonomy never exceeds its authority. The next step for this project is automating the gateway lifecycle and expanding protocol support to MCP, bringing us closer to a universal security layer for the agentic era.
---
title: "From Adapter to Actor: Architectural Patterns for Integrating AI in Your Application"
slug: "ai-integration-patterns-adapter-actor"
translationKey: "llm-integration-patterns-adapter-actor"
date: 2025-12-13T08:00:00+01:00
draft: false
tags:
  - AI
  - LLM
  - Software-Architecture
  - DDD
  - Domain-Driven-Design
  - Hexagonal-Architecture
  - Integration-Patterns
  - Agents
categories:
  - Architecture
  - Development
  - AI
description: "How to distinguish between using an LLM as infrastructure or treating an agent as an actor in your system defines the integration architecture and the level of coupling."
author: "Pascual Montesinos"
---

How you integrate an LLM into your application fundamentally depends on **who has control**: does your code decide the flow while the LLM just responds, or does the LLM decide what to do next?

This distinction leads you to two very different architectural patterns.

## Pattern 1: LLM as Infrastructure

When the interaction is simple—you send a prompt, you receive a response—the LLM is **infrastructure**. It's just another external service, like a translation service or geocoding.

From a hexagonal architecture perspective:

- The domain defines a port (interface)
- The LLM is an adapter that implements that port
- It's interchangeable: OpenAI, Claude, Ollama, a mock for tests

```python
# Port in the domain
class TextAnalyzer(ABC):
    def analyze_sentiment(self, text: str) -> Sentiment:
        ...

# Adapter in infrastructure
class OpenAISentimentAnalyzer(TextAnalyzer):
    def __init__(self, client: OpenAI):
        self.client = client
    
    def analyze_sentiment(self, text: str) -> Sentiment:
        response = self.client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": f"Analyze sentiment: {text}"}]
        )
        return self._parse_response(response)
```

The domain doesn't know there's an LLM behind it. It only knows that something analyzes sentiment. Tomorrow you could swap it for a classic ML model or hardcoded rules and the domain wouldn't notice.

**Characteristics of this pattern:**

- Your code has control of the flow
- The LLM has no state or memory between calls
- No autonomy: prompt in, response out
- Synchronous and predictable in behavior

This pattern also covers cases like RAG (Retrieval Augmented Generation). Even if you enrich the prompt with context from your database, the structure remains the same: your code orchestrates, the LLM responds.

## Pattern 2: Agent as Actor

But there's a qualitative leap when the agent has **rich behavior**: it can query system state, execute commands, make decisions about what to do next.

At that point, the agent stops being infrastructure and becomes an **actor** in your system. It should be treated as you would treat any other external consumer: a user, a B2B partner, another microservice.

The fundamental difference is the **direction of dependency**:

| Pattern 1 | Pattern 2 |
|-----------|-----------|
| Your system → LLM | Agent → Your system |
| Your code initiates | The agent initiates |
| No autonomy | With autonomy |
| Implementation | Actor |

### Variant A: Agent scoped to a service

If the agent has a specific purpose within a bounded context, it can live inside the service and communicate via internal command/query bus. It could also interact with application services directly, though it would be more coupled and would probably require handling transactions.

```python
class OnboardingAgent:
    def __init__(
        self,
        llm: LLMClient,
        command_bus: CommandBus,
        query_bus: QueryBus,
    ):
        self.llm = llm
        self.tools = [
            Tool(
                name="check_customer_exists",
                fn=lambda id: query_bus.ask(GetCustomerQuery(id))
            ),
            Tool(
                name="validate_tax_id", 
                fn=lambda tax_id: query_bus.ask(ValidateTaxIdQuery(tax_id))
            ),
            Tool(
                name="create_customer",
                fn=lambda data: command_bus.execute(CreateCustomerCommand(**data))
            ),
        ]
    
    def run(self, request: OnboardingRequest) -> OnboardingResult:
        return self.llm.run_agent(
            goal=f"Onboard customer: {request.company_name}",
            tools=self.tools,
            context=request.to_dict()
        )
```

The agent has autonomy—it decides which tools to use and in what order—but **respects the boundaries**. It uses the same command bus and query bus that any other component would use. It has no privileged access to repositories or internal services.

**Advantages:**

- Lower latency (no HTTP involved)
- We avoid implementing APIs specifically for an internal agent
- Easier to test (you can mock the bus)

### Variant B: Broad-purpose agent

When the agent needs to interact with multiple services, it should communicate through public interfaces: REST APIs, MCP (Model Context Protocol), or similar.

Here the agent is an external actor. Its tools are wrappers over public APIs:

```python
# The agent's tools call public APIs
tools = [
    MCPTool(
        name="get_order_status",
        endpoint="https://orders.internal/api/orders/{id}",
        method="GET"
    ),
    MCPTool(
        name="cancel_order",
        endpoint="https://orders.internal/api/orders/{id}/cancel",
        method="POST"
    ),
    MCPTool(
        name="process_refund",
        endpoint="https://payments.internal/api/refunds",
        method="POST"
    ),
]
```

The agent knows nothing about the internal implementation of Orders or Payments. It only knows their public contracts.

**Advantages:**

- Total decoupling
- The agent can evolve independently
- Same security/authorization policies as any other client
- Easy to replace or scale

## Architectural Implications

### Coupling

If you give the agent direct access to domain services or repositories, you're coupling your internal implementation to the agent's behavior. Every internal refactor can break the tools.

If the agent uses public interfaces (whether HTTP APIs or command/query buses), you can evolve the internals without affecting it.

### Testing

With Pattern 1, you test the adapter like any other adapter: mocking the HTTP client.

With Pattern 2, you have two levels of testing:

1. **Agent tests**: mock the tools and verify the agent makes the right decisions
2. **Integration tests**: verify the tools work against the real APIs

```python
def test_agent_creates_customer_when_validation_passes():
    # Arrange
    mock_tools = {
        "check_customer_exists": lambda id: None,  # Doesn't exist
        "validate_tax_id": lambda tax_id: {"valid": True},
        "create_customer": Mock()
    }
    agent = OnboardingAgent(llm=mock_llm, tools=mock_tools)
    
    # Act
    agent.run(OnboardingRequest(company_name="Acme", tax_id="B12345678"))
    
    # Assert
    mock_tools["create_customer"].assert_called_once()
```

## When to Use Each Pattern

**Use LLM as infrastructure when:**

- You need a specific capability: summarize, classify, extract, generate
- The flow is predefined in your code

**Use Agent as actor when:**

- The agent decides what actions to take
- It needs to query state and execute commands
- The flow is not predictable in advance

And within Pattern 2:

- **Scoped variant** if the agent operates within a single bounded context
- **Broad variant** if it needs to orchestrate multiple services

## Conclusion

Next time you integrate AI into your application, ask yourself: who has control of the flow?

If your code orchestrates and the LLM just responds, it's infrastructure. Treat it as an adapter.

If the agent decides what to do, it's an actor. Treat it as you would treat any other consumer of your system: through public interfaces and well-defined contracts.

> An agent is not an implementation, it's an actor that interacts with our system.
---
title: "Domain Events vs Integration Events"
slug: "domain-vs-integration-events"
translationKey: "domain-vs-integration-events"
date: 2025-11-21T08:00:00+01:00
draft: false
tags:
  - DDD
  - Domain-Driven-Design
  - Event-Driven-Architecture
  - Integration-Patterns
  - Software-Architecture
  - Bounded-Context
  - Microservices
categories:
  - Architecture
  - Development
  - Software-Design
description: "How distinguishing between domain and integration events in event-driven architectures protects your model while enabling flexible integration between bounded contexts."
author: "Pascual Montesinos"
---

In event-driven architectures, it's useful to distinguish between events based on their purpose: communication within the domain versus integration between bounded contexts. This distinction helps better manage coupling and system evolution.

## The Context

When everything is modeled as "an event," tensions emerge. A single message attempts to serve two different purposes:
- Communicate domain facts to coordinate reactions within the bounded context.
- Expose state changes as an integration contract with other bounded contexts.

For example, when a product is put on sale in an e-commerce system:
- In the domain we need: update projections, recalculate recommendations, update search indexes.
- In integration we need: synchronize product changes with the rest of the bounded contexts.

If we use the same event for both cases, internal domain changes affect integration contracts. And the needs of other bounded contexts add weight to our domain model.

## Two Types of Events, Two Purposes

**Domain events:** Represent facts that occurred in the domain. They are specific, expressive, and optimized for bounded context cohesion. They communicate domain **intent**.

**Integration events:** Represent facts exposed as an integration contract. They are generic, stable, and designed for other bounded contexts to react autonomously. They communicate **observable state**.

### Alternative Nomenclatures

This distinction appears in the literature under different names, though the underlying concept is the same:

- **Internal Events vs External Events**: Emphasizes the audience (inside/outside the system)
- **Private Events vs Public Events**: Emphasizes the contract and visibility

Regardless of the name, the key distinction is: **events that are part of your domain model** versus **events that are part of your integration contract**.

## Naming Convention

To make the distinction evident in code:

**Domain events:** Past tense verbs, ubiquitous domain language

```python
ProductPutOnSale
OrderConfirmed
StockRecalculated
```

**Integration events:** `IntegrationEvent` suffix, generic verbs

```python
ProductUpdatedIntegrationEvent
OrderConfirmedIntegrationEvent
InventoryModifiedIntegrationEvent
```

The key difference:
- `ProductPutOnSale` communicates **what happened in the domain** ("we put on sale")
- `ProductUpdatedIntegrationEvent` communicates **what changed observably** ("the product changed")

When the domain concept is shared and accepted by all bounded contexts (like `OrderConfirmed`), the integration event can directly reflect what happened in the domain: `OrderConfirmedIntegrationEvent`."

Mapping examples:
```
ProductPutOnSale (domain)     → ProductUpdatedIntegrationEvent
OrderConfirmed (domain)       → OrderConfirmedIntegrationEvent
StockRecalculated (domain)    → InventoryModifiedIntegrationEvent
```

## Strategy: Publish Integration as Reaction to Domain

The clearest implementation I've found is to publish integration events as a reaction to domain events:

1. The use case executes its logic and publishes a domain event
2. A handler reacts to that domain event by publishing the corresponding integration event

```python
class PutProductOnSale(UseCase):  
    def __init__(self, repository: ProductRepository, event_producer: DomainEventProducer):  
        self._repository = repository  
        self._event_producer = event_producer  
  
    def execute(self, product_id: str) -> None:  
        product = self._repository.find(product_id)  
        product.put_on_sale()  
  
        self._repository.save(product)  
  
        self._event_producer.publish(
            ProductPutOnSale(product_id=product.id, occurred_at=datetime.now())
        )


class PublishUpdatedProductIntegrationEventOnProductPutOnSale(DomainEventHandler):  
    def __init__(self, query_bus: QueryBus, integration_event_producer: IntegrationEventProducer):  
        self._query_bus = query_bus  
        self._integration_event_producer = integration_event_producer  
  
    def handle(self, event: ProductPutOnSale) -> None:  
        product = self._querybus.ask(
            GetProductQuery(product_id=event.product_id)
        )  
  
        details = self._query_bus.ask(
            GetProductDetailsQuery(product_id=event.product_id)
        )  
  
        self._integration_event_producer.publish(
            ProductUpdatedIntegrationEvent.from_product_details(product, details)
        )
```

The use case only loads what's necessary to put the product on sale (pure domain), while the integration handler executes a specific query to obtain additional information that other bounded contexts require.

## Observed Benefits

**Domain protection:** The domain model doesn't get contaminated with integration needs. The use case remains expressive and focused on the bounded context's business logic.

**Independent evolution:** Domain events can change aggressively when refactoring the model. Integration events maintain compatibility as external contracts. You can split `ProductPutOnSale` into `ProductActivated` + `ProductMadeAvailable` without affecting external consumers.

**Extensibility within the bounded context:** Adding new domain reactions is done by creating new handlers without modifying the original use case.

```python
class ProjectProductOnPutProductOnSale(DomainEventHandler):  
    def __init__(self, project_product: ProjectProduct) -> None:  
        self._project_product = project_product  
    
    def handle(self, event: ProductPutOnSale) -> None:  
        self._project_product.execute(product_id=event.product_id)  
  
  
class RecalculateRecommendationsOnPutProductOnSale(DomainEventHandler):  
    def __init__(self, recalculate_recommendations: RecalculateRecommendations) -> None:  
        self._recalculate_recommendations = recalculate_recommendations  
  
    def handle(self, event: ProductPutOnSale) -> None:  
        self._recalculate_recommendations.execute(product_id=event.product_id)
```

**Predictable latency:** Domain operations focus on what's essential for the operation to occur without penalties for obtaining additional information, without emitting events to centralized systems with higher latency, and without performing side-effects in the use case itself.

**Integration resilience:** If integration publication fails temporarily, a retry mechanism ensures that the integration event will be published as soon as communication with the centralized system is available. This way, the domain remains consistent regardless of integration event delivery to other systems.

## Trade-offs

**Code indirection:** You can't see directly in the use case which integration events are published. This is compensated with distributed tracing and log correlation that allow tracing from the original use case to side-effects produced in other bounded contexts.

**Data re-fetching:** The integration handler needs to query data that the use case had already loaded. In practice, this added cost is usually low, as systems like PostgreSQL leverage their cache to serve these reads very efficiently.

**Conceptual complexity:** Requires explaining and maintaining the distinction between domain and integration events. It's easier to say "we publish an event," but that initial simplicity is paid for with growing coupling between bounded contexts.

## Common Mistakes

**Publishing both from the use case:** Seems more direct, but the use case accumulates knowledge about other bounded contexts. Each new integration requires modifying existing use cases and contaminates the domain.

**Domain events too generic:** Using `ProductUpdated` as a domain event loses expressiveness. `ProductPutOnSale`, `ProductRestocked`, `ProductDiscontinued` communicate intent and allow specific reactions.

**Integration events too specific:** If `ProductPutOnSaleIntegrationEvent` only serves one consumer, you're coupling bounded contexts. Integration events should be useful for multiple consumers.

**Using the same infrastructure for both types:** Domain events and integration events have different infrastructure requirements.

## Infrastructure: Two Systems, Two Purposes

The conceptual distinction should be reflected in the infrastructure:

**Domain Events:** System close to the bounded context, with low latency
- Deployed in the same cluster/infrastructure
- Very low latency (< 10ms typically)
- Transactional guarantees with the aggregate
- Examples: RabbitMQ in the same cluster, outbox pattern with polling, etc.

**Integration Events:** Centralized system between services
- Infrastructure shared between bounded contexts
- Moderate but predictable latency
- Examples: Google Cloud Pub/Sub, Kafka, centralized RabbitMQ

This physical separation has direct impacts:

**Performance:** Use cases complete quickly because they only interact with nearby infrastructure. Publication to external systems occurs asynchronously.

**Resilience:** If the centralized system fails, your bounded context continues functioning. Integration events accumulate locally and are forwarded when it recovers.

**Evolution:** You can change the domain bus implementation without affecting external contracts. Changing the integration system requires coordination between teams.

## Conclusion

This distinction between domain and integration events is an architectural response to a fundamental problem: **coupling grows when a single mechanism tries to serve audiences with divergent contexts**.

Domain events are part of your model. They represent the ubiquitous language, communicate intent, and allow the domain to react to its own changes. Integration events are your asynchronous API. They represent stable contracts, communicate observable state, and allow other bounded contexts to react autonomously. This separation isn't just about brokers or serialization. It's about protecting your domain model while enabling flexible integration. It's about having two speeds of change: fast for your domain, slow for your contracts.

I like to think of domain events as "the internal dialogue of my bounded context" and integration events as "what I communicate to other contexts." One is fast, specific, optimized for cohesion. The other is stable, complete, designed for integration. This isn't a technical decision about infrastructure. It's a decision about where to place your system's boundaries and how you want it to evolve.
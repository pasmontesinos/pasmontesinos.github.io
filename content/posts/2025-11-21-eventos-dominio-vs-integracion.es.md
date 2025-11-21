---
title: "Eventos de Dominio vs Eventos de Integración"
slug: "eventos-dominio-vs-integracion"
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
description: "Cómo distinguir entre eventos de dominio e integración en arquitecturas event-driven protege tu modelo mientras permite integración flexible entre bounded contexts."
author: "Pascual Montesinos"
---

En arquitecturas event-driven, es útil distinguir entre eventos según su propósito: comunicación dentro del dominio versus integración entre bounded contexts. Esta distinción ayuda a manejar mejor el acoplamiento y la evolución del sistema.

## El Contexto

Cuando todo se modela como "un evento", aparecen tensiones. Un mismo mensaje intenta servir a dos propósitos diferentes:
- Comunicar hechos del dominio para coordinar reacciones dentro del bounded context.
- Exponer cambios de estado como contrato de integración con otros bounded contexts.

Por ejemplo, cuando un producto se pone a la venta en un e-commerce:
- En el dominio necesitamos: actualizar proyección, recalcular recomendaciones, actualizar índices de búsqueda.
- En integración necesitamos: sincronizar con el resto de bounded contexts los cambios del producto.

Si usamos el mismo evento para ambos casos, los cambios internos del dominio afectan contratos de integración. Y las necesidades de otros bounded contexts añaden peso a nuestro modelo de dominio.

## Dos Tipos de Eventos, Dos Propósitos

**Eventos de dominio:** Representan hechos que ocurrieron en el dominio. Son específicos, expresivos, y optimizados para la cohesión del bounded context. Comunican **intención** del dominio.

**Eventos de integración:** Representan hechos expuestos como contrato de integración. Son genéricos, estables, y diseñados para que otros bounded contexts reaccionen de forma autónoma. Comunican **estado observable**.

### Nomenclaturas Alternativas

Esta distinción aparece en la literatura con diferentes nombres, aunque el concepto subyacente es el mismo:

- **Eventos Internos vs Eventos Externos**:  Enfatiza la audiencia (dentro/fuera del sistema)
- **Eventos Privados vs Eventos Públicos**:  Enfatiza el contrato y la visibilidad

Independientemente del nombre, la distinción clave es: **eventos que forman parte de tu modelo de dominio** versus **eventos que forman parte de tu contrato de integración**.

## Convención de Naming

Para que la distinción sea evidente en el código:

**Eventos de dominio:** Verbos en pasado, lenguaje ubicuo del dominio

```python
ProductPutOnSale
OrderConfirmed
StockRecalculated
```

**Eventos de integración:** Sufijo `IntegrationEvent`, verbos genéricos

```python
ProductUpdatedIntegrationEvent
OrderConfirmedIntegrationEvent
InventoryModifiedIntegrationEvent
```

La diferencia clave:
- `ProductPutOnSale` comunica **qué pasó en el dominio** ("pusimos a la venta")
- `ProductUpdatedIntegrationEvent` comunica **qué cambió de forma observable** ("el producto cambió")

Cuando el concepto de dominio es compartido y aceptado por todos los bounded contexts (como `OrderConfirmed`), el evento de integración puede reflejar directamente lo que pasó en el dominio: `OrderConfirmedIntegrationEvent`."

Ejemplos del mapeo:

```
ProductPutOnSale (dominio)    → ProductUpdatedIntegrationEvent
OrderConfirmed (dominio)      → OrderConfirmedIntegrationEvent
StockRecalculated (dominio)   → InventoryModifiedIntegrationEvent
```

## Estrategia: Publicar Integración Como Reacción al Dominio

La implementación que he encontrado más clara es publicar eventos de integración como reacción a eventos de dominio:

1. El caso de uso ejecuta su lógica y publica un evento de dominio
2. Un handler reacciona a ese evento de dominio publicando el evento de integración correspondiente

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

El caso de uso solo carga lo necesario para poner el producto a la venta (dominio puro), mientras que el handler de integración ejecuta una query específica para obtener la información adicional que otros bounded contexts requieren.

## Beneficios Observados

**Protección del dominio:** El modelo de dominio no se contamina con necesidades de integración. El caso de uso permanece expresivo y enfocado en la lógica de negocio del bounded context.

**Evolución independiente:** Los eventos de dominio pueden cambiar agresivamente al refactorizar el modelo. Los eventos de integración mantienen compatibilidad como contratos externos. Puedes dividir `ProductPutOnSale` en `ProductActivated` + `ProductMadeAvailable` sin afectar consumidores externos.

**Extensibilidad dentro del bounded context:** Añadir nuevas reacciones de dominio se hace creando nuevos handlers sin modificar el caso de uso original.

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

**Latencia predecible:** Las operaciones de dominio se centran en lo indispensable para que la operación ocurra sin penalizaciones por obtener información adicional, sin emitir eventos en sistemas centralizados con mayor latencia y sin realizar side-effects en el propio use case.

**Resiliencia de integración:** Si la publicación de integración falla temporalmente, un mecanismo de retry asegura que el evento de integración se publicará tan pronto como la comunicación con el sistema centralizado esté disponible. De este modo, el dominio permanece consistente independientemente de la entrega de eventos de integración a otros sistemas.

## Trade-offs

**Indirección en el código:** No puedes ver directamente en el caso de uso qué eventos de integración se publican. Esto se compensa con distributed tracing y correlación de logs que permiten la traza desde el caso de uso original hasta los side-effects producidos en otros bounded contexts.

**Re-fetching de datos:** El handler de integración necesita consultar datos que el caso de uso ya había cargado.  En la práctica, este coste añadido suele ser bajo, ya que sistemas como PostgreSQL aprovechan su caché para servir estas lecturas de forma muy eficiente.

**Complejidad conceptual:** Requiere explicar y mantener la distinción entre eventos de dominio e integración. Es más fácil decir "publicamos un evento", pero esa simplicidad inicial se paga con acoplamiento creciente entre bounded contexts.

## Errores Comunes

**Publicar ambos desde el caso de uso:** Parece más directo, pero el caso de uso acumula conocimiento sobre otros bounded contexts. Cada nueva integración requiere modificar casos de uso existentes y contamina el dominio.

**Eventos de dominio demasiado genéricos:** Usar `ProductUpdated` como evento de dominio pierde la expresividad. `ProductPutOnSale`, `ProductRestocked`, `ProductDiscontinued` comunican intención y permiten reacciones específicas.

**Eventos de integración demasiado específicos:** Si `ProductPutOnSaleIntegrationEvent` solo sirve para un consumidor, estás acoplando bounded contexts. Los eventos de integración deben ser útiles para múltiples consumidores.

**Usar la misma infraestructura para ambos tipos:** Los eventos de dominio y los eventos de integración tienen requisitos de infraestructura diferentes.

## Infraestructura: Dos Sistemas, Dos Propósitos

La distinción conceptual debe reflejarse en la infraestructura:

**Eventos de Dominio:** Sistema cercano al bounded context, con baja latencia
- Desplegado en el mismo cluster/infraestructura
- Latencia muy baja (< 10ms típicamente)
- Garantías transaccionales con el agregado
- Ejemplos: RabbitMQ en el mismo cluster, outbox pattern con polling, etc.

**Eventos de Integración:** Sistema centralizado entre servicios
- Infraestructura compartida entre bounded contexts
- Latencia moderada pero predecible
- Ejemplos: Google Cloud Pub/Sub,  Kafka, RabbitMQ centralizado

Esta separación física tiene impactos directos:

**Rendimiento:** Los casos de uso completan rápido porque solo interactúan con infraestructura cercana. La publicación a sistemas externos ocurre de forma asíncrona.

**Resiliencia:** Si el sistema centralizado falla, tu bounded context sigue funcionando. Los eventos de integración se acumulan localmente y se reenvían cuando se recupere.

**Evolución:** Puedes cambiar la implementación del bus de dominio sin afectar contratos externos. Cambiar el sistema de integración requiere coordinación entre equipos.

## Conclusión

Esta distinción entre eventos de dominio e integración es una respuesta arquitectural a un problema fundamental: **el acoplamiento crece cuando un solo mecanismo intenta servir a audiencias con contextos divergentes**.

Los eventos de dominio son parte de tu modelo. Representan el lenguaje ubicuo, comunican intención, y permiten que el dominio reaccione a sus propios cambios. Los eventos de integración son tu API asíncrona. Representan contratos estables, comunican estado observable, y permiten que otros bounded contexts reaccionen de forma autónoma. Esta separación no es solo sobre brokers o serialización. Es sobre proteger tu modelo de dominio mientras permites integración flexible. Es sobre tener dos velocidades de cambio: una rápida para tu dominio, una lenta para tus contratos.

Me gusta pensar en eventos de dominio como "el diálogo interno de mi bounded context" y eventos de integración como "lo que comunico a otros contextos". Uno es rápido, específico, optimizado para cohesión. El otro es estable, completo, diseñado para integración. Esta no es una decisión técnica sobre infraestructura. Es una decisión sobre dónde colocar las fronteras de tu sistema y cómo quieres que evolucione.

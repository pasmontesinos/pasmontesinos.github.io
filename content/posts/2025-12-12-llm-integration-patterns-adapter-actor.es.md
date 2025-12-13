---
title: "Del adapter al actor: patrones arquitectónicos para integrar IA en tu aplicación"
slug: "patrones-integracion-ia-adapter-actor"
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
description: "Cómo distinguir entre usar un LLM como infraestructura o tratar un agente como actor de tu sistema define la arquitectura de integración y el nivel de acoplamiento."
author: "Pascual Montesinos"
---

La forma en que integras un LLM en tu aplicación depende fundamentalmente de **quién tiene el control**: ¿tu código decide el flujo y el LLM solo responde, o el LLM decide qué hacer a continuación?

Esta distinción te lleva a dos patrones arquitectónicos muy diferentes.

## Patrón 1: LLM como Infraestructura

Cuando la interacción es sencilla —envías un prompt, recibes una respuesta— el LLM es **infraestructura**. Es un servicio externo más, como un servicio de traducción o de geocoding.

Desde la perspectiva de arquitectura hexagonal:

- El dominio define un puerto (interfaz)
- El LLM es un adapter que implementa ese puerto
- Es intercambiable: OpenAI, Claude, Ollama, un mock para tests

```python
# Puerto en el dominio
class TextAnalyzer(ABC):
    def analyze_sentiment(self, text: str) -> Sentiment:
        ...

# Adapter en infraestructura
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

El dominio no sabe que hay un LLM detrás. Solo sabe que existe algo que analiza sentimiento. Mañana podrías cambiarlo por un modelo de ML clásico o por reglas hardcodeadas y el dominio no se enteraría.

**Características de este patrón:**

- Tu código tiene el control del flujo
- El LLM no tiene estado ni memoria entre llamadas
- No hay autonomía: prompt entra, respuesta sale
- Es síncrono y predecible en el comportamiento

Este patrón cubre también casos como RAG (Retrieval Augmented Generation). Aunque enriquezcas el prompt con contexto de tu base de datos, la estructura es la misma: tu código orquesta, el LLM responde.

## Patrón 2: Agente como Actor

Pero hay un salto cualitativo cuando el agente tiene **comportamiento rico**: puede consultar el estado del sistema, ejecutar comandos, tomar decisiones sobre qué hacer a continuación.

En ese momento, el agente deja de ser infraestructura y se convierte en un **actor** de tu sistema. Debe tratarse como tratarías a cualquier otro consumidor externo: un usuario, un partner B2B, otro microservicio.

La diferencia fundamental es la **dirección de la dependencia**:

| Patrón 1 | Patrón 2 |
|----------|----------|
| Tu sistema → LLM | Agente → Tu sistema |
| Tu código inicia | El agente inicia |
| Sin autonomía | Con autonomía |
| Implementación | Actor |

### Variante A: Agente acotado a un servicio

Si el agente tiene un propósito específico dentro de un bounded context, puede vivir dentro del servicio y comunicarse via command/query bus interno. También podría interactuar con servicios de aplicación directamente, aunque estaría más acoplado y probablemente requeriría manejar transacciones.

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

El agente tiene autonomía —decide qué tools usar y en qué orden— pero **respeta los boundaries**. Usa el mismo command bus y query bus que usaría cualquier otro componente. No tiene acceso privilegiado a repositorios o servicios internos.

**Ventajas:**

- Menor latencia (no hay HTTP de por medio)
- Evitamos implementar APIs específicas para un agente interno
- Más fácil de testear (puedes mockear el bus)

### Variante B: Agente de propósito amplio

Cuando el agente necesita interactuar con múltiples servicios, debería comunicarse a través de interfaces públicas: APIs REST, MCP (Model Context Protocol), o similar.

Aquí el agente es un actor externo. Sus tools son wrappers sobre las APIs públicas:

```python
# Las tools del agente llaman a APIs públicas
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

El agente no sabe nada de la implementación interna de Orders o Payments. Solo conoce sus contratos públicos.

**Ventajas:**

- Desacoplamiento total
- El agente puede evolucionar independientemente
- Mismas políticas de seguridad/autorización que cualquier otro cliente
- Fácil de reemplazar o escalar

## Implicaciones arquitectónicas

### Acoplamiento

Si le das al agente acceso directo a servicios de dominio o repositorios, estás acoplando tu implementación interna al comportamiento del agente. Cada refactor interno puede romper las tools.

Si el agente usa interfaces públicas (ya sean APIs HTTP o command/query buses), puedes evolucionar los internos sin afectarle.

### Testing

Con el Patrón 1, testeas el adapter como cualquier otro adapter: mockeando el cliente HTTP.

Con el Patrón 2, tienes dos niveles de testing:

1. **Tests del agente**: mockeas las tools y verificas que el agente toma las decisiones correctas
2. **Tests de integración**: verificas que las tools funcionan contra las APIs reales

```python
def test_agent_creates_customer_when_validation_passes():
    # Arrange
    mock_tools = {
        "check_customer_exists": lambda id: None,  # No existe
        "validate_tax_id": lambda tax_id: {"valid": True},
        "create_customer": Mock()
    }
    agent = OnboardingAgent(llm=mock_llm, tools=mock_tools)
    
    # Act
    agent.run(OnboardingRequest(company_name="Acme", tax_id="B12345678"))
    
    # Assert
    mock_tools["create_customer"].assert_called_once()
```

## Cuándo usar cada patrón

**Usa LLM como infraestructura cuando:**

- Necesitas una capacidad específica: resumir, clasificar, extraer, generar
- El flujo está predefinido en tu código

**Usa Agente como actor cuando:**

- El agente decide qué acciones tomar
- Necesita consultar estado y ejecutar comandos
- El flujo no es predecible de antemano

Y dentro del Patrón 2:

- **Variante acotada** si el agente opera dentro de un único bounded context
- **Variante amplia** si necesita orquestar múltiples servicios

## Conclusión

La próxima vez que integres IA en tu aplicación, pregúntate: ¿quién tiene el control del flujo?

Si tu código orquesta y el LLM solo responde, es infraestructura. Trátalo como un adapter.

Si el agente decide qué hacer, es un actor. Trátalo como tratarías a cualquier otro consumidor de tu sistema: a través de interfaces públicas y contratos bien definidos.

> Un agente no es una implementación, es un actor que interactúa con nuestro sistema.
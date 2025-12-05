---
title: "Domain Instrumentation: Manteniendo los Casos de Uso Expresivos"
slug: "domain-instrumentation"
translationKey: "domain-instrumentation"
date: 2025-12-05T00:00:00+01:00
draft: false
tags:
  - DDD
  - Domain-Driven-Design
  - Testing
  - Clean-Code
  - Software-Architecture
  - Observability
  - Metrics
  - Logging
categories:
  - Architecture
  - Development
  - Software-Design
  - Testing
description: "Abstraer la instrumentación detrás de interfaces específicas de dominio mantiene los casos de uso expresivos, enfocados en la lógica de negocio y dramáticamente más fáciles de testear."
author: "Pascual Montesinos"
---

La instrumentación es esencial en cualquier aplicación: logs para debugging, métricas para monitorización y trazas para entender el flujo de ejecución. Pero cuando esta instrumentación se mezcla directamente en los casos de uso, el código se vuelve difícil de leer, mantener y especialmente, de testear.

En este post exploraré un patrón que habitualmente aplico: **abstraer la instrumentación detrás de interfaces específicas de dominio**, manteniendo los casos de uso expresivos y enfocados en la lógica de negocio. Además, me facilita posponer decisiones de infraestructura y mejora dramáticamente la calidad de los tests.

## El Problema: Casos de Uso Contaminados

Consideremos un caso de uso para cancelar pedidos en un sistema de e-commerce:

```python
import time
import logging
from metrics import Counter, Summary

logger = logging.getLogger(__name__)

orders_cancelled_total = Counter(
    'orders_cancelled_total',
    'Total orders cancelled',
    ['status', 'result']
)

order_cancellation_duration = Summary(
    'order_cancellation_duration_seconds',
    'Time spent cancelling orders',
    ['result']
)

class CancelOrderUseCase:
    def __init__(self, order_repository: OrderRepository):
        self._order_repository = order_repository
    
    def execute(self, order_id: OrderId) -> None:
        logger.info(f"Starting order cancellation for order {order_id}")
        start_time = time.time()
        
        order = self._order_repository.get(order_id)
        if order is None:
            logger.warning(f"Order {order_id} not found")
            duration = time.time() - start_time
            orders_cancelled_total.labels(status='unknown', result='not_found').inc()
            order_cancellation_duration.labels(result='not_found').observe(duration)
            return
        
        logger.info(f"Order {order_id} found, status: {order.status}")
        
        order.cancel()
        self._order_repository.save(order)
        
        duration = time.time() - start_time
        logger.info(f"Order {order_id} cancelled successfully in {duration:.2f}s")
        orders_cancelled_total.labels(
            status=order.status.value,
            result='success'
        ).inc()
        order_cancellation_duration.labels(result='success').observe(duration)
```

Este código funciona, pero tiene varios problemas:

1. **Mezcla responsabilidades**: Lógica de negocio con logging, métricas y timing
2. **Difícil de leer**: Cuesta distinguir qué es esencial y qué es instrumentación
3. **Repetición de lógica de timing**: El cálculo de duración se repite en múltiples lugares
4. **Testing complicado**

Los tests correspondientes muestran la complejidad:

```python
from unittest.mock import Mock, patch, call
import pytest

class TestCancelOrderUseCase:
    def test_cancel_order_successfully(self):
        order_id = OrderId("123")
        order = Mock(spec=Order)
        order.id = order_id
        order.status = OrderStatus.PENDING
        
        order_repository = Mock(spec=OrderRepository)
        order_repository.get.return_value = order
        
        use_case = CancelOrderUseCase(order_repository)
        
        orders_cancelled_total.clear()
        order_cancellation_duration.clear()
        
        with patch('time.time', side_effect=[100.0, 100.5]):
            with patch('your_module.logger') as mock_logger:
                use_case.execute(order_id)
        
        order.cancel.assert_called_once()
        order_repository.save.assert_called_once_with(order)
        
        mock_logger.info.assert_has_calls([
            call("Starting order cancellation for order 123"),
            call("Order 123 found, status: PENDING"),
            call("Order 123 cancelled successfully in 0.50s"),
        ])
        
        counter_value = orders_cancelled_total.labels(
            status='PENDING',
            result='success'
        )._value.get()
        assert counter_value == 1.0
        
        summary_value = order_cancellation_duration.labels(
            result='success'
        )._sum.get()
        assert summary_value == 0.5

    def test_cancel_order_not_found(self):
        order_id = OrderId("999")
        
        order_repository = Mock(spec=OrderRepository)
        order_repository.get.return_value = None
        
        use_case = CancelOrderUseCase(order_repository)
        
        orders_cancelled_total.clear()
        order_cancellation_duration.clear()
        
        with patch('time.time', side_effect=[100.0, 100.3]):
            with patch('your_module.logger') as mock_logger:
                use_case.execute(order_id)
        
        mock_logger.warning.assert_called_once_with("Order 999 not found")
        
        counter_value = orders_cancelled_total.labels(
            status='unknown',
            result='not_found'
        )._value.get()
        assert counter_value == 1.0
        
        summary_value = order_cancellation_duration.labels(
            result='not_found'
        )._sum.get()
        assert summary_value == 0.3
```

El problema es evidente: los tests son verbosos, frágiles y oscurecen lo que realmente se está testeando: **la lógica de negocio**. Se verifican strings de logs, se accede a internals de las métricas, se mockea `time.time()`, y se limpian métricas globales.

## La Solución: Interface Específica de Dominio

El cambio clave es usar **el lenguaje ubicuo del dominio** en la interface de instrumentación:

```python
from abc import ABC, abstractmethod

class OrderInstrumentation(ABC):
    @abstractmethod
    def canceling_order(self, order_id: OrderId) -> None: ...
    
    @abstractmethod
    def order_found(self, order: Order) -> None: ...
    
    @abstractmethod
    def order_not_found(self, order_id: OrderId) -> None: ...
    
    @abstractmethod
    def order_cancelled(self, order: Order) -> None: ...

class CancelOrderUseCase:
    def __init__(
        self,
        order_repository: OrderRepository,
        instrumentation: OrderInstrumentation,
    ):
        self._order_repository = order_repository
        self._instrumentation = instrumentation
    
    def execute(self, order_id: OrderId) -> None:
        self._instrumentation.canceling_order(order_id)
        
        order = self._order_repository.get(order_id)
        if order is None:
            self._instrumentation.order_not_found(order_id)
            raise OrderNotFound(order_id)
        
        self._instrumentation.order_found(order)
        
        order.cancel()
        self._order_repository.save(order)
        
        self._instrumentation.order_cancelled(order)
```

El cambio es significativo:

1. **El timing desapareció del caso de uso**: La instrumentación se encarga internamente
2. **No hay strings mágicos**: Todo está expresado en términos del dominio
3. **No hay métricas globales**: La instrumentación es inyectada
4. **El código es más legible**: Se lee como una narración de lo que está pasando

Y lo más importante: **los tests se vuelven expresivos**:

```python
class TestCancelOrderUseCase:
    def test_cancel_order_successfully(self):
        order_id = OrderId("123")
        order = Order(id=order_id, status=OrderStatus.PENDING)
        
        order_repository = Mock(spec=OrderRepository)
        order_repository.get.return_value = order
        
        instrumentation = Mock(spec=OrderInstrumentation)
        
        use_case = CancelOrderUseCase(order_repository, instrumentation)
        
        use_case.execute(order_id)
        
        order_repository.save.assert_called_once()
        assert order.status == OrderStatus.CANCELLED
        
        instrumentation.canceling_order.assert_called_once_with(order_id)
        instrumentation.order_found.assert_called_once_with(order)
        instrumentation.order_cancelled.assert_called_once_with(order)

    def test_cancel_order_not_found(self):
        order_id = OrderId("999")
        
        order_repository = Mock(spec=OrderRepository)
        order_repository.get.return_value = None
        
        instrumentation = Mock(spec=OrderInstrumentation)
        
        use_case = CancelOrderUseCase(order_repository, instrumentation)
        
        with pytest.raises(OrderNotFound):
            use_case.execute(order_id)
        
        instrumentation.canceling_order.assert_called_once_with(order_id)
        instrumentation.order_not_found.assert_called_once_with(order_id)
        instrumentation.order_cancelled.assert_not_called()
```

El salto de calidad es claro:

- No más mocks de `time.time()`
- No más verificaciones de strings
- No más acceso a internals de las métricas
- No más limpieza de métricas globales
- Los tests hablan el lenguaje del dominio
- Son más cortos, claros y expresivos

## Implementación: De Abstracción a Infraestructura

La implementación de `OrderInstrumentation` usa las piezas de infraestructura reales (logging y métricas):

```python
import logging
from abc import ABC, abstractmethod
from metrics import Counter, Summary

logger = logging.getLogger(__name__)

class Clock(ABC):
    @abstractmethod
    def now(self) -> float: ...

class SystemClock(Clock):
    def now(self) -> float:
        return time.time()

class LoggingOrderInstrumentation(OrderInstrumentation):
    def __init__(
        self,
        counter: Counter,
        summary: Summary,
        clock: Clock,
    ):
        self._counter = counter
        self._summary = summary
        self._clock = clock
        self._operation_start_times: dict[str, float] = {}
    
    def canceling_order(self, order_id: OrderId) -> None:
        logger.info(f"Starting order cancellation for order {order_id}")
        self._operation_start_times[str(order_id)] = self._clock.now()
    
    def order_found(self, order: Order) -> None:
        logger.info(f"Order {order.id} found, status: {order.status}")
    
    def order_not_found(self, order_id: OrderId) -> None:
        duration = self._calculate_duration(order_id)
        
        logger.warning(f"Order {order_id} not found")
        self._counter.labels(status='unknown', result='not_found').inc()
        self._summary.labels(result='not_found').observe(duration)
        
        self._cleanup_timing(order_id)
    
    def order_cancelled(self, order: Order) -> None:
        duration = self._calculate_duration(order.id)
        
        logger.info(f"Order {order.id} cancelled successfully in {duration:.2f}s")
        self._counter.labels(
            status=order.status.value,
            result='success'
        ).inc()
        self._summary.labels(result='success').observe(duration)
        
        self._cleanup_timing(order.id)
    
    def _calculate_duration(self, order_id: OrderId) -> float:
        start_time = self._operation_start_times.get(str(order_id))
        if start_time is None:
            return 0.0
        return self._clock.now() - start_time
    
    def _cleanup_timing(self, order_id: OrderId) -> None:
        self._operation_start_times.pop(str(order_id), None)
```

**Separación de responsabilidades en el testing:**

La implementación tiene sus propios tests unitarios que verifican que la instrumentación produce la telemetría correcta:

```python
class TestLoggingOrderInstrumentation:
    def test_order_instrumentation_records_success_metrics(self): ...
    def test_order_instrumentation_records_not_found_metrics(self): ...
    def test_order_instrumentation_logs_messages(self): ...
```

Estos tests sí acceden a internals de las métricas y mockean el `Clock`, pero están aislados en la capa de infraestructura. Los tests del caso de uso permanecen limpios y expresivos.

## Beneficios y Principios

Este patrón trae múltiples beneficios:

**1. Casos de uso expresivos y enfocados**
El código habla en términos del dominio, sin distracciones de infraestructura.

**2. Tests dramáticamente mejores**
- Más cortos y legibles
- Verifican comportamiento, no implementación
- Sin mocks de infraestructura (logger, time, métricas)
- Lenguaje de dominio en las assertions

**3. Separation of Concerns**
La instrumentación es una responsabilidad separada, con su propia lógica y tests.

**4. Dependency Inversion**
Los casos de uso dependen de abstracciones (`OrderInstrumentation`), no de detalles técnicos (`Counter`, `Summary`, `Logger`).

**5. Lenguaje Ubicuo hasta en la instrumentación**
Los métodos `order_cancelled()`, `canceling_order()` son parte del lenguaje del dominio, haciendo el código más expresivo.

**6. Timing sin contaminar el dominio**
La medición de duración está encapsulada en la implementación, el caso de uso no necesita saberlo.

**7. Métricas inyectadas, no globales**
Facilita el testing y evita estado compartido global.

## Conclusión

La comparación antes/después muestra la mejora:

**Antes:**
```python
logger.info(f"Starting order cancellation for order {order_id}")
start_time = time.time()
# ... lógica
duration = time.time() - start_time
logger.info(f"Order cancelled in {duration:.2f}s")
orders_cancelled_total.labels(status=order.status.value, result='success').inc()
order_cancellation_duration.labels(result='success').observe(duration)
```

**Después:**
```python
self._instrumentation.canceling_order(order_id)
# ... lógica
self._instrumentation.order_cancelled(order)
```

La instrumentación es esencial, pero no debe contaminar los casos de uso. Al abstraerla detrás de interfaces específicas de dominio, se consigue código más limpio, expresivo y especialmente, **mucho más fácil de testear**.

El testing es un driver poderoso del buen diseño. Cuando los tests son complicados, verbosos o frágiles, están indicando que algo en el diseño necesita mejorar. En este caso, nos llevan a descubrir un patrón que separa responsabilidades, mantiene el código expresivo y alineado con el lenguaje ubicuo del dominio.
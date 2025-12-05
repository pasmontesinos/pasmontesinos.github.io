---
title: "Domain Instrumentation: Keeping Use Cases Expressive"
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
description: "Abstracting instrumentation behind domain-specific interfaces keeps use cases expressive, focused on business logic, and dramatically easier to test."
author: "Pascual Montesinos"
---

Instrumentation is essential in any application: logs for debugging, metrics for monitoring, and traces to understand execution flow. But when this instrumentation is mixed directly into use cases, the code becomes difficult to read, maintain, and especially, to test.

In this post, I'll explore a pattern I regularly apply: **abstracting instrumentation behind domain-specific interfaces**, keeping use cases expressive and focused on business logic. Additionally, it helps me postpone infrastructure decisions and dramatically improves test quality.

## The Problem: Contaminated Use Cases

Consider a use case for canceling orders in an e-commerce system:

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

This code works, but has several problems:

1. **Mixed responsibilities**: Business logic with logging, metrics, and timing
2. **Hard to read**: It's difficult to distinguish what's essential from what's instrumentation
3. **Timing logic repetition**: Duration calculation is repeated in multiple places
4. **Complicated testing**

The corresponding tests show the complexity:

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

The problem is evident: tests are verbose, fragile, and obscure what's really being tested: **the business logic**. We're verifying log strings, accessing metrics internals, mocking `time.time()`, and clearing global metrics.

## The Solution: Domain-Specific Interface

The key change is using **the ubiquitous language of the domain** in the instrumentation interface:

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

The change is significant:

1. **Timing disappeared from the use case**: Instrumentation handles it internally
2. **No magic strings**: Everything is expressed in domain terms
3. **No global metrics**: Instrumentation is injected
4. **Code is more readable**: It reads as a narrative of what's happening

And most importantly: **tests become expressive**:

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

The quality leap is clear:

- No more `time.time()` mocks
- No more string verifications
- No more accessing metrics internals
- No more clearing global metrics
- Tests speak the domain language
- They're shorter, clearer, and more expressive

## Implementation: From Abstraction to Infrastructure

The `OrderInstrumentation` implementation uses the real infrastructure pieces (logging and metrics):

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

**Separation of responsibilities in testing:**

The implementation has its own unit tests that verify the instrumentation produces the correct telemetry:

```python
class TestLoggingOrderInstrumentation:
    def test_order_instrumentation_records_success_metrics(self): ...
    def test_order_instrumentation_records_not_found_metrics(self): ...
    def test_order_instrumentation_logs_messages(self): ...
```

These tests do access metrics internals and mock the `Clock`, but they're isolated in the infrastructure layer. Use case tests remain clean and expressive.

## Benefits and Principles

This pattern brings multiple benefits:

**1. Expressive and focused use cases**
Code speaks in domain terms, without infrastructure distractions.

**2. Dramatically better tests**
- Shorter and more readable
- Verify behavior, not implementation
- No infrastructure mocks (logger, time, metrics)
- Domain language in assertions

**3. Separation of Concerns**
Instrumentation is a separate responsibility, with its own logic and tests.

**4. Dependency Inversion**
Use cases depend on abstractions (`OrderInstrumentation`), not on technical details (`Counter`, `Summary`, `Logger`).

**5. Ubiquitous Language even in instrumentation**
Methods like `order_cancelled()`, `canceling_order()` are part of the domain language, making code more expressive.

**6. Timing without contaminating the domain**
Duration measurement is encapsulated in the implementation; the use case doesn't need to know about it.

**7. Injected metrics, not globals**
Facilitates testing and avoids shared global state.

## Conclusion

The before/after comparison shows the improvement:

**Before:**
```python
logger.info(f"Starting order cancellation for order {order_id}")
start_time = time.time()
# ... logic
duration = time.time() - start_time
logger.info(f"Order cancelled in {duration:.2f}s")
orders_cancelled_total.labels(status=order.status.value, result='success').inc()
order_cancellation_duration.labels(result='success').observe(duration)
```

**After:**
```python
self._instrumentation.canceling_order(order_id)
# ... logic
self._instrumentation.order_cancelled(order)
```

Instrumentation is essential, but it shouldn't contaminate use cases. By abstracting it behind domain-specific interfaces, we achieve cleaner, more expressive code that's especially **much easier to test**.

Testing is a powerful driver of good design. When tests are complicated, verbose, or fragile, they're indicating that something in the design needs improvement. In this case, they lead us to discover a pattern that separates concerns, keeps code expressive, and aligned with the domain's ubiquitous language.
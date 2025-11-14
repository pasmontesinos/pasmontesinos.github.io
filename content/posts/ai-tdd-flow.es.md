---
title: "Manteniendo la disciplina en AI-Assisted TDD"
date: 2025-11-14T08:00:00+01:00
draft: false
slug: "ai-assisted-tdd-flujo"
translationKey: "ai-assisted-tdd-flow"
tags:
  - TDD
  - AI-Assisted-Development
  - Test-Driven-Development
  - LLM
  - Software-Development
  - Best-Practices
categories:
  - Development
  - Testing
  - AI
description: "Cómo practicar Test-Driven Development con asistentes de IA manteniendo la disciplina del ciclo TDD mientras aprovechamos las capacidades de los LLMs"
author: "Pascual Montesinos"
---

Después de varios meses experimentando con AI-assisted TDD, he ido refinando un flujo de trabajo que me permite combinar la disciplina estricta del TDD tradicional con las capacidades de los asistentes de IA. No se trata de acelerar el proceso a cualquier coste, sino de mantener la calidad y el control mientras el agente se encarga de las tareas más mecánicas.

## El desafío de no perder el norte

Uno de los principales problemas al trabajar con asistentes de IA en TDD es mantener la disciplina del proceso. La tentación es real: dejar que el agente genere tanto tests como implementación de una sola vez parece eficiente. Pero al hacerlo, perdemos los beneficios fundamentales del TDD. Perdemos el diseño emergente guiado por tests, la implementación mínima necesaria, y esa confianza que te da el proceso cuando necesitas refactorizar.

Los LLMs son extraordinariamente capaces de generar código completo y bien estructurado. Pero esa misma capacidad se convierte en una trampa si no establecemos límites claros. El código que "funciona de primera" muchas veces anticipa funcionalidad que aún no necesitamos, incluye abstracciones prematuras, o simplemente hace más de lo que el test actual pide.

Por eso es crucial un flujo que mantiene estas características del TDD clásico mientras aprovecha lo que los asistentes hacen bien: generar código acotado, seguir patrones establecidos, y ejecutar las tareas más repetitivas.

## Separación clara de responsabilidades

El flujo se basa en una premisa simple: roles claramente definidos entre el desarrollador y el agente de IA. Yo, como desarrollador, mantengo el control de las decisiones. Decido qué test implementar a continuación, si la implementación propuesta es verdaderamente mínima o está anticipando funcionalidad, si un refactor es adecuado o no es el momento oportuno.

El agente, por su parte, se encarga de proponer el código de los tests siguiendo las especificaciones que le doy, implementar la solución mínima bajo mi dirección explícita, ejecutar el test runner y reportar resultados, identificar oportunidades de refactoring una vez estamos en verde, y sugerir mensajes de commit que sigan convenciones semánticas.

Esta separación no es arbitraria. Replica el flujo natural del pair programming, donde uno escribe código mientras el otro revisa y guía. La diferencia es que el agente tiene una capacidad de generación de código mucho mayor que un humano, pero también una tendencia natural a sobre-implementar si no se le guía con firmeza.

## Comenzando con una lista de tests

Antes de entrar en el ciclo TDD propiamente dicho, el agente me propone una lista de tests que refinamos y que sirve como mapa durante la sesión de desarrollo. Esta práctica, que detallo en [mi post anterior sobre la lista de tests](https://pasmontesinos.github.io/posts/ai-assisted-tdd-lista-de-tests/), es fundamental para mantener el contexto y la dirección del desarrollo.

Por ejemplo, cuando empiezo a implementar un repositorio de productos con Django ORM, la lista inicial podría verse así:

```
Lista de Tests - ProductRepository
===================================
1. ⌛ test_should_save_product
2. ⌛ test_should_find_existing_product
3. ⌛ test_should_return_none_when_product_not_found
```

Esta lista evoluciona constantemente. Descubrimos nuevos casos que no habíamos considerado, eliminamos tests que resultan redundantes, ajustamos prioridades según aprendemos más del dominio. Pero lo importante es que siempre sabemos dónde estamos y hacia dónde vamos.

## Configurando el agente: el archivo AGENTS.md

La clave para mantener la disciplina está en documentar explícitamente el flujo. En cada proyecto, incluyo un archivo `AGENTS.md` en la raíz con las instrucciones precisas que el agente debe seguir durante el ciclo TDD:

```markdown
## Ciclo estricto de trabajo TDD (con agente):

**1. Selección del Test**
- El agente muestra la lista de tests actualizada y espera a que el humano elija el test a implementar.

**2. Fase Rojo**
- El agente propone el código de test correspondiente y el humano lo acepta o propone cambios.
- Una vez aceptado, el agente modifica el código (sin comentarios).
- El agente lanza los tests con el comando especificado en el contexto del repositorio y verifica que **falla por el motivo esperado**.
- Si el fallo no es el esperado, el agente ajusta el test hasta lograr el fallo correcto.
- **No se realiza commit en esta fase.**

**3. Fase Verde**
- El agente propone la **implementación mínima** necesaria para que el test pase, y el humano la acepta o propone cambios. Es MUY IMPORTANTE implementar solo lo necesario para pasar el test actual y no anticiparse a tests futuros.
- Una vez aceptado, el agente modifica el código (sin comentarios).
- El agente lanza los tests para verificar que **todos los tests pasan correctamente**.
- Si algún test falla, el agente propone correcciones hasta que todos pasen.
- El agente propone un commit con el mensaje `<feat|chore|test|refactor|...>: <breve descripción en inglés en una línea>` con el comando especificado en el contexto del repositorio y pregunta: *"¿Hago el commit ahora?"*

**4. Fase Refactor**
- El agente propone **mejoras seguras de refactorización** (sin cambiar el comportamiento observable).
- Una vez aceptado por el humano, el agente modifica el código (sin comentarios).
- El agente lanza los tests y confirma que todos siguen pasando.
- Si algún test falla, el agente propone correcciones hasta que todos pasen.
- El agente propone un commit con el mensaje `<feat|chore|test|refactor|...>: <breve descripción en inglés en una línea>` con el comando especificado en el contexto del repositorio y pregunta: *"¿Hago el commit ahora?"*

El ciclo se repite hasta completar la funcionalidad. Es imprescindible mostrar la lista actualizada en cada iteración para mantener el contexto del avance.
```

## El ciclo: Rojo, Verde, Refactor

Ahora veamos cada fase con más profundidad y ejemplos concretos de cómo funciona en la práctica.

### Selección consciente del test

El proceso comienza siempre con una selección consciente del siguiente test a implementar. El agente presenta la lista actualizada y yo elijo cuál abordar. Esta decisión no es trivial: considero la complejidad incremental, las dependencias entre tests, etc. No se trata simplemente de hacer "el siguiente" en la lista, sino de elegir el test correcto en el momento correcto.

Cuando le digo al agente "implementemos el test 1", estamos estableciendo un contrato: vamos a enfocarnos en esta funcionalidad específica, ni más ni menos.

### Fase Rojo: el fallo correcto

El agente propone el código del test. Por ejemplo:

```python
def test_should_save_product(self) -> None:
    repository = DjangoORMProductRepository()
    product = Product(
        id="product-1",
        name="Laptop",
        price=Decimal("999.99"),
        stock=10
    )
    
    repository.save(product)
    
    product_model = ProductModel.objects.get(id=product.id)
    assert product_model.id == product.id
    assert product_model.name == product.name
    assert product_model.price == product.price
    assert product_model.stock == product.stock
```

Reviso el test. ¿Está bien estructurado? ¿Las aserciones son las correctas? Si algo no me convence, lo ajustamos. Una vez aprobado, el agente ejecuta el test.

Aquí viene algo crítico: el test debe fallar por la razón correcta. Si falla porque el método `save()` no existe, ese es el fallo esperado. Pero si falla por un error de sintaxis, un import faltante, o un problema de configuración, no estamos realmente en rojo. Estamos en un estado intermedio inválido. El agente ajusta hasta conseguir el fallo esperado.

### Fase Verde: la tentación de hacer más

Una vez tenemos el test fallando por el motivo correcto, llega el momento de la implementación. Y aquí es donde la disciplina se pone más a prueba. Le pido al agente que proponga la implementación mínima para pasar el test.

El agente propone algo como:

```python
class DjangoORMProductRepository:
    def save(self, product: Product) -> None:
        ProductModel.objects.create(
            id=product.id,
            name=product.name,
            price=product.price,
            stock=product.stock
        )
```

Mi trabajo es revisar que realmente sea mínima. ¿Está implementando solo el método `save()`? ¿No está añadiendo métodos `find()` o `search()` anticipándose a futuros tests? ¿No está agregando validaciones que aún no hemos testeado? Si la implementación es verdaderamente mínima, le doy luz verde.

El agente modifica el código y ejecuta todos los tests. No solo el nuevo, todos. Esta es otra práctica fundamental: cada cambio debe verificarse contra toda la suite de tests para asegurar que no hemos roto nada existente.

Una vez todos los tests pasan, el agente propone un commit: "feat: implement save method in ProductRepository". Reviso el mensaje, confirmo que sigue las convenciones de conventional commits, y aprobamos. En ese momento, el agente valida los tipos y el formato antes de hacer commit. Ahora sí, el código está en el repositorio.

### Fase Refactor: mejorando con confianza

Con los tests en verde, llega el momento de preguntarnos si hay oportunidades de mejora. El agente identifica posibles refactorings: nombres que podrían ser más claros, duplicación que podría extraerse, estructura que podría simplificarse, etc. Yo decido si es el momento adecuado para aplicar estos cambios o si deben esperar.

Si tiene sentido hacerlo ahora, el agente aplica los cambios y vuelve a ejecutar todos los tests. Si algo se rompe, los tests nos lo dicen inmediatamente. Si todo sigue verde, hacemos otro commit: "refactor: extract domain model conversion to private method".

Esta confianza para refactorizar es uno de los principales beneficios del TDD. No estamos adivinando si nuestros cambios rompen algo. Los tests nos dan la respuesta de forma inmediata y objetiva.

## El ritmo emergente

Después de varios ciclos, emerge un ritmo natural. La lista de tests se va actualizando:

```
Lista de Tests - ProductRepository
===================================
1. ✅ test_should_save_product
2. ❌ test_should_find_existing_product
3. ⌛ test_should_return_none_when_product_not_found
```

Cada símbolo cuenta una historia. Los tests completados (✅) nos dan confianza. El test en rojo (❌) nos muestra dónde estamos. Los pendientes (⌛) nos recuerdan hacia dónde vamos. Este feedback visual es sorprendentemente útil.

## Lo que he ganado con este flujo

El control del proceso se mantiene completamente en mis manos. Yo decido qué implementar y cuándo. El agente ejecuta, pero yo dirijo. Esto es fundamental porque el contexto del negocio, las prioridades del producto, los trade-offs técnicos, todo eso requiere juicio humano que no quiero dejar en manos del agente.

La implementación es verdaderamente incremental. Al forzar la implementación mínima en cada ciclo, el diseño emerge sin sobre-ingeniería. No estoy construyendo para un futuro hipotético, estoy construyendo para necesidades reales y presentes.

Los commits son atómicos y significativos. Cada ciclo produce uno o dos commits pequeños y enfocados. Cuando necesito hacer un rollback, sé exactamente qué estoy revirtiendo. Cuando un compañero revisa el código, puede entender la progresión lógica del desarrollo. El historial de Git se convierte en documentación del proceso de diseño.

Y hay un beneficio inesperado: aprendo continuamente. Al revisar las propuestas del agente, descubro nuevos patrones, formas diferentes de estructurar el código, convenciones que no conocía. El agente se convierte en un compañero de aprendizaje, no solo una herramienta de productividad.

## Integrando con el equipo

Este flujo no está pensado para trabajar en aislamiento. De hecho, funciona especialmente bien en contextos colaborativos. En pair programming, uno puede manejar el agente mientras el otro revisa y sugiere dirección. En mob programming, el agente se convierte en una herramienta más del mob, donde el driver opera el agente bajo la guía del grupo.

## Reflexiones finales

He llegado a entender que el AI-assisted TDD no es sobre escribir código más rápido. Es sobre mantener la calidad mientras delegamos lo mecánico. El agente es excepcional generando pequeños fragmentos de código, siguiendo patrones establecidos, ejecutando comandos repetitivos. Pero el criterio, las decisiones de diseño, la comprensión del dominio de negocio, todo eso sigue siendo responsabilidad del desarrollador.

La clave está en no ceder el control del proceso. El agente es una herramienta poderosa, probablemente la herramienta más potente que he usado en años de desarrollo. Pero sigue siendo eso: una herramienta. La dirección, las decisiones, el "por qué" detrás del código, eso sigue siendo territorio humano.

Y quizás lo más importante: este flujo me permite mantener esa sensación de ownership sobre el código. Cuando algo sale a producción, entiendo cada decisión que se tomó. Puedo explicar por qué está estructurado así, qué alternativas se consideraron, qué trade-offs se aceptaron. No es código que un agente generó y yo aprobé ciegamente. Es código que diseñé, con un asistente que me ayudó a materializarlo.

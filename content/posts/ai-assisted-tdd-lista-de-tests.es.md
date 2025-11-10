---
title: "Lista de Tests como guía en AI-Assisted TDD"
slug: "ai-assisted-tdd-lista-de-tests"
translationKey: "ai-assisted-tdd-test-list"
date: 2025-01-10T12:00:00+01:00
draft: false
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
description: "Cómo la lista de tests de Kent Beck cobra nueva relevancia al practicar TDD con asistentes de IA, proporcionando contexto y control durante el ciclo de desarrollo."
author: "Pascual Montesinos"
---

# Lista de Tests como guía en AI-Assisted TDD

## Introducción

La práctica de **AI-assisted TDD** (Test-Driven Development asistido por inteligencia artificial) combina los principios fundamentales del TDD con las capacidades de los modelos de lenguaje de gran escala (LLMs).  
En este enfoque, el desarrollador colabora con un asistente de IA durante el ciclo de desarrollo, aprovechando su capacidad para generar código y mantener el flujo iterativo característico del TDD.

Uno de los principales retos al trabajar con asistentes de IA es mantener la coherencia y la dirección del proceso. En este contexto, resulta especialmente relevante recuperar un concepto introducido por **Kent Beck** hace más de dos décadas: la **lista de tests** como guía del desarrollo.

## La Lista de Tests según Kent Beck

En su libro *Test Driven Development: By Example*, Kent Beck destaca la importancia de mantener una **lista de tareas pendientes** durante el proceso de TDD.  
Esta lista ayuda a conservar el foco, registrar ideas que surgen durante el desarrollo y mantener el control sobre el ritmo y la dirección del trabajo.

Beck presenta esta práctica desde las primeras páginas del libro, utilizándola para anotar tests pendientes, refactorizaciones necesarias o ideas que aparecen durante el ciclo.  
La regla es sencilla: mantener una lista y utilizarla como ancla para no perder el foco; si algo interrumpe el flujo actual, se anota en la lista y se continúa con la tarea en curso.

En el contexto del desarrollo asistido por IA, esta práctica adquiere una relevancia renovada. La lista no solo permite conservar el contexto de trabajo, sino que además ofrece al asistente una referencia clara del estado actual del desarrollo y de los próximos pasos posibles.

## Un Prompt para Mantener el Contexto

La siguiente estructura de prompt permite mantener una lista de tests actualizada durante cada iteración del ciclo TDD:

```markdown
Durante el ciclo TDD, el agente y el humano trabajarán con una **Lista de Tests** para mantener el contexto del avance.

Es IMPRESCINDIBLE mostrar la lista de tests actualizada en cada iteración para que el humano pueda elegir el siguiente test a implementar.

**Lista de Tests:**
- Se mostrará como una lista numerada en Markdown.
- Cada ítem seguirá el formato:  
  `<número>. <icono_estado> <TestClass>.test_should_<descripción> — <objetivo breve>`
- Iconos de estado:  
  - ✅ si el test ya está implementado y pasa.  
  - ❌ si el test está implementado pero actualmente falla.  
  - ⌛ si el test todavía no está implementado.  
- Las entradas deben ser cortas e incrementales.  
- No se muestra código, solo la lista.  
- El humano indicará el siguiente test indicando su número.
```

## Ejemplo de Implementación

Con este enfoque, una sesión típica de AI-assisted TDD puede estructurarse del siguiente modo:

```
Lista de tests actual:

1. ✅ TestDjangoORMProductRepository.test_should_save_product — persiste un producto nuevo en la base
2. ❌ TestDjangoORMProductRepository.test_should_find_existing_product — recupera un producto previamente guardado
3. ⌛ TestDjangoORMProductRepository.test_should_return_none_when_product_not_found — responde None si no existe

Propondría el commit G: implement Django product repository save. ¿Hago el commit ahora?

Siguiente paso natural: 1) implementar el test 2 para la búsqueda; 2) luego cubrir la ausencia (test 3).
```

## Beneficios Observados

### 1. **Control del Proceso**
La lista visual ofrece una representación clara del progreso y de los tests pendientes, lo que facilita la toma de decisiones sobre qué abordar a continuación.

### 2. **Comunicación Eficiente**
En lugar de explicar el siguiente paso en detalle, basta con indicar “implementemos el test 2”, y el asistente comprende el contexto exacto y la tarea a realizar.

### 3. **Recuperación de Contexto**
Cuando se reanuda una sesión después de una interrupción, la lista permite reconstruir rápidamente el estado del desarrollo, tanto para el desarrollador como para el asistente.

### 4. **Decisiones Estratégicas**
La lista facilita priorizar según la complejidad, el tiempo disponible o las dependencias entre funcionalidades.

## Reflexiones Finales

El AI-assisted TDD representa una evolución natural del TDD tradicional, que mantiene sus principios fundamentales y aprovecha las capacidades de los LLM modernos.  
La lista de tests actúa como un puente entre la disciplina del TDD y el potencial generativo de la inteligencia artificial.

Esta práctica permite conservar el control del proceso de desarrollo mientras se incrementa la productividad y la consistencia.  
Como recordaba Kent Beck, la clave sigue siendo la misma: **no olvides tu lista de tests**.

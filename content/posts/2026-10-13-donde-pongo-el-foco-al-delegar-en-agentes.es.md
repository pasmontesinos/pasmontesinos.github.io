---
title: "Dónde pongo el foco cuando delego en agentes"
slug: "donde-pongo-el-foco-al-delegar-en-agentes"
translationKey: "where-i-focus-when-delegating-to-agents"
date: 2026-10-13T08:00:00+02:00
draft: false
tags:
  - AI-Assisted-Development
  - Agents
  - LLM
  - TDD
  - Continuous-Delivery
  - Lean
  - XP
categories:
  - Development
  - AI
description: "Cómo pasé de dirigir agentes en tiempo real a un loop donde yo decido la spec y el merge, y el agente trabaja siguiendo mis reglas."
---

Hace casi un año escribí sobre [cómo mantener la disciplina en AI-assisted TDD](https://pasmontesinos.com/es/posts/ai-assisted-tdd-flujo/). En aquel flujo dirigía al agente en tiempo real, en la misma sesión en la que escribía el código: elegía cada test, corregía el diseño sobre la marcha y aprobaba cada commit.

Ese flujo fue cambiando de forma gradual. Los modelos mejoraban de una versión a otra, y las convenciones del repositorio se iban afinando con cada corrección que dejaba por escrito. Con ambas cosas, cada vez tenía menos sentido dirigir la implementación paso a paso y más sentido poner el peso en la spec: dejar claro qué construir y cómo, y delegar el resto. Llegó un punto en que la implementación iba en piloto automático hasta quedar lista para revisión.

Pero yo seguía en medio, saltando de una sesión de Claude Code a otra. Pasar la spec, lanzar la implementación, pedir la review, aplicar los comentarios, arrancar la siguiente slice. Todo a golpe de prompt repetitivo, sesión tras sesión.

[grooveloop](https://grooveloop.pasmontesinos.com) nació para sustituir esa coordinación. Y una vez existía, otras cosas llegaron casi solas: rondas de corrección automáticas, slices independientes en paralelo y lecciones de la review incorporadas a las convenciones.

| | Antes | Ahora |
| --- | --- | --- |
| Decisiones de diseño | En tiempo real, mientras el agente escribe | En la spec, antes de empezar |
| Correcciones | Sobre la marcha, en la sesión | En la review, y convertidas en convenciones |
| Implementación | Dirigida paso a paso | En piloto automático hasta la PR |
| Coordinación entre pasos | Yo, a golpe de prompt | El loop |

Este post no va de la herramienta, sino de **dónde ha quedado mi atención** después de ese cambio. No es una receta ni una promesa de productividad. Es lo que me está funcionando a mí, con sus límites. Si ya trabajas con agentes pero sigues dirigiéndolos sesión a sesión, quizá te sirva para dar el siguiente paso: delegar más sin perder el control y con menos fricción.

## El flujo, en una imagen

![El ciclo de una slice: brainstorming y spec, sub-issue, developer abre PR, revisor automático, mi review, siguiente slice o PR de convenciones](/images/posts/grooveloop-flujo.es.svg)

El punto de partida es una Story ya pensada y cortada en slices; cómo llego a ella lo cuento más abajo. Desde ahí el loop crea una sub-issue por slice y lanza un agente que la implementa en un worktree aislado y abre una PR. Otro agente la revisa antes que yo. Yo decido: merge, cerrar o pedir otra ronda. Con el merge se desbloquean las slices siguientes, y las independientes avanzan en paralelo.

## Dónde pongo el énfasis

Antes mi atención estaba dentro de la sesión, en tiempo real. Ahora está en los extremos: en la spec y las convenciones, antes de que el agente empiece, y en la review, antes de que nada llegue a master. En medio, piloto automático.

### 1. La spec es el prompt

El flujo no empieza escribiendo una spec, sino conversando. Parto de una user story o una tarea y abro una sesión de brainstorming con el agente. Ahí profundizamos en cómo abordar el cambio: arquitectura, diseño, estrategia de delivery y cualquier duda que pueda acabar en un malentendido.

Solo cuando eso está claro escribo la Story. Con ayuda de la skill de slicing, la subo como Issue con el formato que entiende el loop y las slices ya cortadas. Es donde más tiempo invierto, con diferencia.

Invertir aquí me ahorra trabajo después. Cada duda que resuelvo antes de implementar es una corrección o una redirección menos cuando el código ya está escrito. Y si el corte es malo, lo pago varias PRs más tarde.

El corte en slices, su orden y sus dependencias los decido yo, y sigo trabajando con **incrementos pequeños**. Cada slice lleva su valor, su alcance, sus criterios de aceptación y una **verificación observable**: qué comprobaciones demuestran que funciona una vez mergeada. Que sean pequeños es lo que mantiene la review manejable y permite llevar cada merge a producción de forma segura.

### 2. El merge es mi decisión

Cada slice termina en una pull request. Antes de que yo la vea, otro agente la ha revisado y ha dejado su veredicto. Pero la decisión es mía: el merge aprueba, cerrar rechaza y un comentario con una directiva pide otra ronda.

El auto-merge existe, pero está apagado por defecto, y es deliberado. Solo lo activo cuando me compensa: repos sin despliegue automático, con protección de rama y donde deshacer es barato.

### 3. Cada corrección se convierte en convención

Esta es la parte que más ha cambiado mi día a día. Cuando corrijo a un agente en una review, puedo marcar esa corrección como lección. Un agente la convierte en una pull request contra las convenciones del repositorio, que reviso como cualquier otra.

La idea es sencilla: no quiero tener que corregir lo mismo dos veces. Es mejora continua aplicada al propio sistema de trabajo, y conecta con algo que ya hacía: documentar convenciones en ficheros planos, con reglas imperativas y un único ejemplo positivo por regla.

### 4. Determinismo donde se puede

Lo de "loop determinista" no significa que los agentes lo sean. Significa que el orquestador es código aburrido: en cada tick lee GitHub, detecta qué ha cambiado y actúa. Cada paso es idempotente. GitHub es la fuente de verdad y no hay broker, base de datos ni webhooks.

Lo no determinista queda encerrado dentro de cada slice, en un worktree aislado, con un número limitado de reintentos y de rondas de corrección. Lo que pasa entre slices, en qué orden y cuándo, no depende de que un agente se acuerde.

## ¿Y el TDD?

Sigue ahí, pero ya no lo piloto test a test. La disciplina que antes imponía yo en cada ciclo ahora está escrita: en la persona del agente que implementa, en las skills de TDD y verificación que carga antes de empezar y en las convenciones del repositorio. El agente revisor y yo comprobamos después que se ha respetado.

He cambiado control fino por control en los puntos que más importan. Lo que antes vigilaba en cada paso, ahora lo vigilo en la spec y en la review.

La sensación de ownership que defendía en aquel post sí ha cambiado, y no quiero maquillarlo. Es inevitable: cuanto más te alejas de escribir el código, menos control tienes sobre él. Ya no conozco cada rincón. Lo que lo compensa es el brainstorming, que me obliga a entrar en el cambio lo suficiente para saber cómo encajan las piezas. Y las convenciones me dan la confianza de que se construyen como deben.

## Por qué "groove"

Soy percusionista. En un grupo, la percusión sostiene el pulso desde dentro y todos se apoyan en él. Eso es el groove: tocar sin fricción. Es lo que buscaba para el delivery: que un cambio fluya de la spec al merge sin quedarse parado entre un paso y el siguiente.

## Reflexiones finales

Lo que me ha dado este loop no es que el código se escriba más rápido. Es que nada se queda esperando a que yo me acuerde, y que mi tiempo va a lo que sigue siendo mío: pensar el cambio, cortarlo bien, decidir qué entra y enseñar al sistema a no repetir errores.

**Yo decido. El agente trabaja para mí, siguiendo mis reglas.**

Si te interesa, tienes más detalles en la [web del proyecto](https://grooveloop.pasmontesinos.com), y el código está en [GitHub](https://github.com/pasmontesinos/grooveloop) y en [PyPI](https://pypi.org/project/grooveloop/). Y si trabajas de forma parecida, o muy distinta, me encantará saber dónde pones tú el foco.

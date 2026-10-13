# Blog de Pascual Montesinos

Notas sobre ingeniería de software, Extreme Programming y Lean Software Development, en español e inglés.

🌐 [pasmontesinos.com](https://pasmontesinos.com)

## Ejecutar en local

El sitio está hecho con [Hugo](https://gohugo.io) y el tema [PaperMod](https://github.com/adityatelange/hugo-PaperMod), que se incluye como submódulo:

```bash
git submodule update --init --recursive
hugo server -D
```

El blog queda en `http://localhost:1313/`. La opción `-D` incluye los borradores.

## Escribir un post

Cada post tiene una versión por idioma en `content/posts/`:

- `AAAA-MM-DD-<slug-es>.es.md`
- `AAAA-MM-DD-<slug-en>.en.md`

Las dos versiones comparten el mismo `translationKey`, en inglés e igual al slug del fichero inglés, para que Hugo las enlace entre sí.

```yaml
---
title: "..."
slug: "..."
translationKey: "..."
date: 2026-10-13T08:00:00+02:00
draft: true
tags: []
categories: []
description: "..."
---
```

Las imágenes van en `static/images/posts/` y se referencian como `/images/posts/<nombre>.<idioma>.svg`.

## Publicar

Al hacer push a `master`, el workflow `.github/workflows/deploy.yml` construye el sitio y lo despliega en GitHub Pages. Solo se publican los posts con `draft: false` y con fecha pasada.

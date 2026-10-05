---
title: "Olá, mundo: recomeçando o blog"
date: 2026-10-05T10:00:00-03:00
tags: [blog, hugo]
description: Primeiro post do blog novo, feito com Hugo e publicado no GitHub Pages.
---

Este é o primeiro post do blog novo. Cada post é um arquivo Markdown dentro de `content/posts/`, e o GitHub Actions publica tudo no GitHub Pages a cada push.

## Como escrever um post

Crie um arquivo em `content/posts/AAAA/` com um *front matter* no topo:

```yaml
---
title: "Título do post"
date: 2026-10-05T10:00:00-03:00
tags: [tag-um, tag-dois]
---
```

> Dica: rode `hugo new content posts/2026/meu-post.md` e o arquivo já nasce com o modelo.

## Traduções

Para publicar a versão em inglês, crie `ola-mundo.en.md` ao lado deste arquivo. O seletor **PT | EN** no topo liga as duas versões automaticamente.

---

Uma lista para ver o estilo:

- Markdown puro
- Busca embutida (`Ctrl K`)
- Modo claro e escuro

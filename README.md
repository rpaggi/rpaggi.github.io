# rpaggi.github.io

Blog pessoal feito com [Hugo](https://gohugo.io) + tema [Hextra](https://github.com/imfing/hextra),
com visual inspirado no [akitaonrails.com](https://akitaonrails.com). Publicado no GitHub Pages
via GitHub Actions a cada push na `main`.

## Escrevendo um post

```sh
hugo new content posts/2026/meu-post.md   # nasce como draft: true
hugo server -D                            # preview em http://localhost:1313 (inclui drafts)
```

Tire o `draft: true` do front matter quando quiser publicar, dê commit e push.

Front matter:

```yaml
---
title: "Título do post"
date: 2026-10-05T10:00:00-03:00
tags: [hugo, blog]
description: "Resumo curto (usado em SEO/RSS)"
---
```

- URL final: `/AAAA/MM/DD/<nome-do-arquivo>/`.
- Post com imagens: use uma pasta `content/posts/2026/meu-post/index.md` e coloque as imagens
  ao lado (`![alt](foto.jpg)`).

## Traduções (PT-BR / EN)

PT-BR é o idioma padrão (raiz `/`), EN fica em `/en/`. A tradução é **outro arquivo** ao lado
do original, com `.en` antes da extensão:

```
content/posts/2026/meu-post.md      → /2026/10/05/meu-post/
content/posts/2026/meu-post.en.md   → /en/2026/10/05/meu-post/
```

Posts sem tradução só aparecem na versão em português; o seletor `PT | EN` fica esmaecido neles.

## Estrutura

```
content/          posts e páginas em Markdown
layouts/          overrides do tema (home, post, listagens, seletor de idioma)
assets/css/       custom.css: paleta, fontes e estilo
i18n/             textos da interface em pt/en
hugo.yaml         configuração (menus, idiomas, permalinks)
```

## Requisitos locais

Hugo **extended** ≥ 0.146 e Go (o tema é baixado como Hugo Module).

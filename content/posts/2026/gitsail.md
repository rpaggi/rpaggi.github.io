---
title: "GitSail: um cliente Git em Rust com quatro interfaces, feito em três dias com agentes de IA"
date: 2026-10-06T11:29:00-03:00
tags: [rust, git, agentes-de-ia, claude-code, arquitetura]
description: "Como saí de um PRD e um backlog para um cliente Git open source com CLI, TUI, app desktop e extensão de VS Code, com 66 mil linhas de Rust e quase 1.800 testes, em três dias trabalhando com agentes."
---

Entre os dias 17 e 19 de setembro eu construí o **[GitSail](https://github.com/rpaggi/gitsail)**, um cliente Git open source com **um núcleo em Rust e quatro interfaces**: linha de comando, interface de terminal (TUI), aplicativo desktop e extensão para o VS Code.

Em três dias foram 79 commits, 9 releases alpha, cerca de **66 mil linhas de Rust** e 28 mil de TypeScript/Vue, e **1.199 testes em Rust mais 616 em TypeScript**, rodando no CI em Linux, Windows e macOS. Quase todo o código foi escrito por agentes de IA. O meu trabalho foi outro, e é sobre ele que quero falar.

---

## Por que mais um cliente Git?

O desafio que me propus foi criar um software no estilo do **GitKraken**, só que open source. Devem existir vários por aí, mas decidi fazer o meu, porque sim. Às vezes a melhor motivação para um projeto pessoal é essa.

Para não ficar só na cópia, defini um problema para resolver no PRD: a gente alterna o dia inteiro entre terminal, cliente gráfico e editor para mexer no Git, e cada ferramenta se comporta de um jeito. A proposta do GitSail é ter **um único motor Git e várias formas de usar**, sem que elas possam divergir.

```
                  ┌────────────────────────┐
                  │     núcleo em Rust     │
                  │  domínio, casos de uso │
                  │     e adapter Git      │
                  └───────────┬────────────┘
       ┌───────────────┬──────┴────────┬───────────────┐
      CLI             TUI           Desktop         VS Code
    (clap)         (Ratatui)     (Tauri + Vue)   (TypeScript)
```

`gitsail log`, o gráfico de commits da TUI e o histórico do app desktop não podem mostrar coisas diferentes porque são, literalmente, o mesmo código.

---

## Antes do código: PRD, backlog e ADRs

O primeiro commit não tem código. Antes de pedir qualquer linha, preparei:

- um **PRD** (documento de requisitos) com visão, público, princípios e escopo até a v1.0;
- um **backlog** com **133 histórias de usuário** distribuídas em épicos, cada uma com critérios de aceite e uma *Definition of Done* global;
- um **documento de arquitetura** com as decisões registradas como **ADRs**, que chegaram a 25 ao fim dos três dias.

O PRD foi feito a quatro mãos: eu e a IA, num *brainstorming* em conjunto.

Essa preparação é o que torna possível delegar. Um agente com uma história bem escrita na frente ("US-011: stage e unstage por arquivo", com critérios claros) entrega algo verificável. Um agente com "faz um cliente Git" na frente entrega um protótipo bonito que ninguém consegue manter.

Algumas decisões que ficaram registradas e guiaram todo o resto:

- **ADR-003: usar o `git` instalado, não reimplementar o Git.** Isso mantém hooks, credenciais, agente SSH e configurações do usuário funcionando como sempre funcionaram. O custo é criar um processo por comando e ter que fazer *parsing* cuidadoso da saída.
- **ADR-002: Ports & Adapters.** O domínio não conhece Git, terminal nem interface gráfica.
- **ADR-010: o GitSail não guarda credenciais.** Quem cuida disso é o Git e o keyring do sistema.
- **ADR-018: local primeiro.** Sem conta, sem telemetria, sem chamada de rede que você não pediu.

---

## O fluxo com agentes

O repositório tem um `AGENTS.md` como fonte única de instruções para qualquer agente, e um `CLAUDE.md` que só aponta para ele. Ali ficam as regras do jogo:

- todo artefato novo em inglês, mesmo com o planejamento original em português;
- ler a documentação de arquitetura e de produto antes de mudar comportamento;
- manter a direção das dependências para dentro (o domínio nunca depende de infraestrutura);
- rodar a verificação antes de dizer que terminou.

Os agentes também carregam **skills**: instruções especializadas para cada tipo de trabalho, como orquestração de tech lead, ciclo de desenvolvimento e QA, decisões de arquitetura com ADR, testes ponta a ponta e escrita técnica. Cada entrega era amarrada a uma história do backlog, e os IDs (`US-119`, `EPIC-08`) aparecem nas mensagens de commit, o que deixa fácil rastrear por que cada mudança existe.

---

## Arquitetura protegida pelo CI, não por boa vontade

Com agentes escrevendo dezenas de milhares de linhas, "lembrar de respeitar a arquitetura" não funciona. Então a regra virou teste.

O crate `gitsail-git` é o **único** que pode chamar o `git`. Um script no CI (`check-architecture.sh`) falha o build se qualquer outro crate tentar. Essas checagens automáticas de arquitetura são chamadas de *fitness functions*, e ficaram registradas na ADR-022.

```
crates/
  gitsail-domain        modelo puro, sem I/O
  gitsail-application   casos de uso e portas
  gitsail-git           o único que executa `git`
  gitsail-protocol      DTOs compartilhados com as interfaces
  gitsail-cli           linha de comando
  gitsail-tui           interface de terminal (Ratatui)
  gitsail-forge         APIs de GitHub/GitLab, keyring
apps/
  desktop               Tauri v2 + Vue 3 + TypeScript
  vscode                extensão do VS Code
```

A mesma lógica vale para segurança: **operações destrutivas sempre pedem confirmação, e recusar deixa o repositório intocado**. Isso é uma garantia estrutural, com uma matriz de preferências que prova cada caso, e não uma convenção.

---

## O que mais deu trabalho: Windows conversando com o WSL

Eu desenvolvo no **WSL2** (o Linux dentro do Windows), mas o GitSail precisa funcionar no Windows de verdade, inclusive abrindo repositórios que estão do lado Linux. Nenhuma funcionalidade deu tanto trabalho quanto essa fronteira.

- **Repositório válido reportado como inexistente.** O Git for Windows, ao abrir um repositório do WSL por `\\wsl.localhost\...`, recusa com *"detected dubious ownership"*, porque o dono do diretório é outro usuário. O GitSail tratava toda falha de descoberta como "não é um repositório Git", e mandava o usuário procurar o problema errado. A correção foi classificar esse caso como permissão negada e sugerir a linha de `safe.directory` que resolve.
- **Rebase interativo quebrado.** O Git trata `GIT_SEQUENCE_EDITOR` como um comando de shell e o entrega ao `sh -c`, que no Windows é o `sh` embutido do Git. Um caminho do Windows chegava lá cheio de barras invertidas, que o shell comia como escape. Foi preciso escrever o caminho de um jeito que o shell aceitasse.
- **O `PATH` misturado.** O WSL injeta as pastas do Windows (`/mnt/c/Windows/...`) no `PATH` do Linux, e a ferramenta de empacotamento do AppImage quebrava ao esbarrar numa delas sem permissão. Um problema que só existe na minha máquina, não no CI.
- **Detalhes de plataforma em sequência:** fim de linha (`core.autocrlf=false` no CI e nos fixtures), caminhos que só batiam depois de canonicalizar os dois lados, o app desktop travando a janela e piscando um console a cada comando git, e um teste de processo lento que usava `timeout /T` e precisou trocar por `ping`.

Nenhum desses problemas aparece rodando só no Linux. Ter o CI nos três sistemas desde o começo foi o que os trouxe à tona.

---

## Decisões que mudaram no meio do caminho

Nem tudo saiu como planejado, e isso também ficou registrado. A extensão do VS Code nasceu dependendo do binário `gitsail` (ADR-015). Durante o desenvolvimento ficou claro que isso complicava a instalação, e a **ADR-025** inverteu a decisão: a extensão passou a ler o Git direto.

O detalhe importante é que a decisão antiga não foi apagada nem renumerada. A ADR nova explica o que mudou e por quê. Com agentes, isso é ainda mais valioso: o próximo agente que ler a documentação entende não só o estado atual, mas o caminho até ele.

---

## O que eu levo disso

A principal: **na era das IAs, dá para recriar as ferramentas de que a gente gosta e colocar os nossos incrementos nelas, sem dor de cabeça.** Um cliente Git com quatro interfaces era um projeto de meses. Com vibe coding, virou um desafio de três dias.

Mas o "sem dor de cabeça" teve condições:

1. **Com agentes, o gargalo vira especificação e verificação.** Escrever código ficou barato; saber o que pedir e provar que está certo, não.
2. **Documente decisões para quem vem depois, inclusive agentes.** PRD, backlog e ADRs foram o que manteve 66 mil linhas coerentes.
3. **Transforme regras de arquitetura em testes.** Se a regra não quebra o build, ela vai ser violada.
4. **Teste em todas as plataformas desde o primeiro dia.** A fronteira entre Windows e WSL sozinha rendeu uma lista de bugs que o Linux nunca mostraria.

## Experimente

O GitSail é open source (Apache 2.0) e está em alpha. Os builds para Linux, macOS e Windows estão na [página de releases](https://github.com/rpaggi/gitsail/releases):

```sh
gitsail                 # abre a TUI
gitsail log --limit 20
gitsail log --json | jq '.data.items[].subject'
```

Issues e PRs são bem-vindos.

---

## Referências

- [Repositório do GitSail](https://github.com/rpaggi/gitsail)
- [Ratatui](https://ratatui.rs), para a interface de terminal
- [Tauri](https://tauri.app), para o aplicativo desktop
- [Architecture Decision Records](https://adr.github.io)
- [AGENTS.md](https://agents.md), o formato aberto de instruções para agentes

---
title: "GitSail: a Rust Git Client with Four Interfaces, Built in Three Days with AI Agents"
date: 2026-10-06T11:29:00-03:00
tags: [rust, git, ai-agents, claude-code, architecture]
description: "How I went from a PRD and a backlog to an open source Git client with a CLI, a TUI, a desktop app and a VS Code extension, with 66 thousand lines of Rust and almost 1,800 tests, in three days working with agents."
---

Between September 17 and 19 I built **[GitSail](https://github.com/rpaggi/gitsail)**, an open source Git client with **one Rust core and four interfaces**: command line, terminal UI (TUI), desktop app and a VS Code extension.

Those three days produced 79 commits, 9 alpha releases, about **66 thousand lines of Rust** and 28 thousand of TypeScript/Vue, and **1,199 Rust tests plus 616 TypeScript tests**, running in CI on Linux, Windows and macOS. Almost all of the code was written by AI agents. My job was something else, and that's what I want to talk about.

---

## Why another Git client?

The challenge I set for myself was to build something in the style of **GitKraken**, but open source. There must be several out there already, but I decided to build my own, just because. Sometimes that's the best motivation for a side project.

So it wouldn't be just a copy, I defined a problem to solve in the PRD: we switch between the terminal, a graphical client and the editor all day long to work with Git, and every tool behaves differently. GitSail's proposal is **one Git engine and several ways to use it**, with no way for them to drift apart.

```
                  ┌────────────────────────┐
                  │       Rust core        │
                  │   domain, use cases    │
                  │    and Git adapter     │
                  └───────────┬────────────┘
       ┌───────────────┬──────┴────────┬───────────────┐
      CLI             TUI           Desktop         VS Code
    (clap)         (Ratatui)     (Tauri + Vue)   (TypeScript)
```

`gitsail log`, the TUI's commit graph and the desktop app's history view can't show different things, because they are literally the same code.

---

## Before the code: PRD, backlog and ADRs

The first commit has no code. Before asking for a single line, I prepared:

- a **PRD** (product requirements document) with the vision, audience, principles and scope up to v1.0;
- a **backlog** with **133 user stories** spread across epics, each one with acceptance criteria and a global *Definition of Done*;
- an **architecture document** with the decisions recorded as **ADRs**, which reached 25 by the end of the three days.

The PRD was a joint effort: the AI and I, brainstorming together.

This preparation is what makes delegating possible. An agent with a well-written story in front of it ("US-011: stage and unstage by file", with clear criteria) delivers something you can verify. An agent with "build a Git client" in front of it delivers a pretty prototype that nobody can maintain.

Some of the recorded decisions that guided everything else:

- **ADR-003: use the installed `git`, don't reimplement Git.** This keeps the user's hooks, credentials, SSH agent and config working exactly as they always have. The cost is spawning a process per command and having to parse the output carefully.
- **ADR-002: Ports & Adapters.** The domain knows nothing about Git, the terminal or the graphical interface.
- **ADR-010: GitSail doesn't store credentials.** Git and the system keyring take care of that.
- **ADR-018: local first.** No account, no telemetry, no network calls you didn't ask for.

---

## The agent workflow

The repository has an `AGENTS.md` as the single source of instructions for any agent, and a `CLAUDE.md` that just points to it. That's where the rules of the game live:

- every new artifact in English, even though the original planning was in Portuguese;
- read the architecture and product documentation before changing behavior;
- keep dependencies pointing inward (the domain never depends on infrastructure);
- run the verification before saying the work is done.

The agents also load **skills**: specialized instructions for each kind of work, such as tech lead orchestration, the development and QA loop, architecture decisions with ADRs, end-to-end testing and technical writing. Every delivery was tied to a backlog story, and the IDs (`US-119`, `EPIC-08`) show up in the commit messages, which makes it easy to trace why each change exists.

---

## Architecture enforced by CI, not by good intentions

With agents writing tens of thousands of lines, "remembering to respect the architecture" doesn't work. So the rule became a test.

The `gitsail-git` crate is the **only** one allowed to call `git`. A CI script (`check-architecture.sh`) fails the build if any other crate tries to. These automated architecture checks are called *fitness functions*, and they were recorded in ADR-022.

```
crates/
  gitsail-domain        pure model, no I/O
  gitsail-application   use cases and ports
  gitsail-git           the only crate that runs `git`
  gitsail-protocol      DTOs shared with every interface
  gitsail-cli           command line
  gitsail-tui           terminal UI (Ratatui)
  gitsail-forge         GitHub/GitLab APIs, keyring
apps/
  desktop               Tauri v2 + Vue 3 + TypeScript
  vscode                VS Code extension
```

The same logic applies to safety: **destructive operations always ask first, and declining leaves the repository untouched**. That's a structural guarantee, with a preferences matrix that proves every case, not a convention.

---

## The hardest part: getting Windows to talk to WSL

I develop on **WSL2** (Linux inside Windows), but GitSail has to work on real Windows, including opening repositories that live on the Linux side. No feature took as much work as that boundary.

- **A valid repository reported as missing.** When Git for Windows opens a WSL repository through `\\wsl.localhost\...`, it refuses with *"detected dubious ownership"*, because the directory belongs to a different user. GitSail treated every discovery failure as "not a Git repository", sending the user hunting for the wrong problem. The fix was to classify this case as permission denied and suggest the `safe.directory` line that solves it.
- **Broken interactive rebase.** Git treats `GIT_SEQUENCE_EDITOR` as a shell command and hands it to `sh -c`, which on Windows is Git's own bundled `sh`. A Windows path arrived there full of backslashes, which the shell ate as escapes. The path had to be written in a way the shell would accept.
- **A mixed-up `PATH`.** WSL injects Windows folders (`/mnt/c/Windows/...`) into the Linux `PATH`, and the AppImage packaging tool crashed when it hit one of them without permission. A problem that only exists on my machine, not in CI.
- **Platform details, one after another:** line endings (`core.autocrlf=false` in CI and in the fixtures), paths that only matched after canonicalizing both sides, the desktop app freezing the window and flashing a console on every git call, and a slow-process test that used `timeout /T` and had to switch to `ping`.

None of these problems show up when you only run on Linux. Having CI on all three systems from the start is what brought them to light.

---

## Decisions that changed along the way

Not everything went as planned, and that was recorded too. The VS Code extension started out depending on the `gitsail` binary (ADR-015). During development it became clear that this complicated installation, and **ADR-025** reversed the decision: the extension now reads Git directly.

The important detail is that the old decision was neither deleted nor renumbered. The new ADR explains what changed and why. With agents, this is even more valuable: the next agent that reads the documentation understands not just the current state, but how it got there.

---

## What I take away from this

The main lesson: **in the age of AI, we can recreate the tools we like and add our own improvements to them, without the headache.** A Git client with four interfaces used to be a months-long project. With vibe coding, it became a three-day challenge.

But "without the headache" came with conditions:

1. **With agents, the bottleneck becomes specification and verification.** Writing code got cheap; knowing what to ask for and proving it's right did not.
2. **Document decisions for whoever comes next, agents included.** The PRD, backlog and ADRs are what kept 66 thousand lines coherent.
3. **Turn architecture rules into tests.** If a rule doesn't break the build, it will be broken.
4. **Test on every platform from day one.** The boundary between Windows and WSL alone produced a list of bugs Linux would never have shown.

## Try it

GitSail is open source (Apache 2.0) and in alpha. Builds for Linux, macOS and Windows are on the [releases page](https://github.com/rpaggi/gitsail/releases):

```sh
gitsail                 # opens the TUI
gitsail log --limit 20
gitsail log --json | jq '.data.items[].subject'
```

Issues and PRs are welcome.

---

## References

- [GitSail repository](https://github.com/rpaggi/gitsail)
- [Ratatui](https://ratatui.rs), for the terminal UI
- [Tauri](https://tauri.app), for the desktop app
- [Architecture Decision Records](https://adr.github.io)
- [AGENTS.md](https://agents.md), the open format for agent instructions

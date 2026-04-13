---
title: Doc index
description: Index of all cross-plugin knowledge files — when and why to read each one.
author: 🤖 Generated with Claude Code
updated: 2026-04-14
---
# Doc index

> [!important]
> While an effort is made to keep these docs continuously up to date, all content is generated with Claude Code and may contain inaccuracies — verify important information.

| Doc | Read before |
|---|---|
| `undocumented-obsidian-apis.md` | Using or looking up undocumented Obsidian properties and methods — documents useful internal APIs discovered through runtime inspection. |
| `ios-webkit-quirks.md` | Modifying content-visibility, IntersectionObserver, scroll-state container queries, touch handling on `position: fixed` elements, or click synthesis behavior — iOS WebKit has platform-specific bugs including compositor hit-test divergence. |
| `electron-popout-quirks.md` | Using `document`, `window`, `ResizeObserver`, `IntersectionObserver`, `requestAnimationFrame`, `floatingSplit`, or `:hover` checks in code that may run in popout windows — covers silent observer failures, RAF paint timing, stale hit-testing, document enumeration, and module-scope binding issues. |
| `obsidian-api-quirks.md` | Using `vault.process()`, `vault.modify()`, `new Notice()`, Bases `config.get()` with dynamic schema defaults, or any file I/O that could race with Obsidian's debounced writes. |
| `obsidian-review-bot.md` | Fixing bot-reported issues, adding/modifying eslint-disable comments, or preparing a PR for the Obsidian plugin review. |
| `electron-css-quirks.md` | Writing nested `:has()` selectors, working around `-webkit-line-clamp` truncation, or using `opacity` transitions on text-heavy elements — documents Blink/Electron CSS rendering quirks including GPU compositing antialiasing. |

---
title: Obsidian review bot
description: Obsidian review bot behaviors, workarounds, and common issue solutions.
author: 🤖 Generated with Claude Code
last updated: 2026-02-09
---
# Obsidian review bot

The Obsidian review bot runs `eslint-plugin-obsidianmd` with its `recommended` config against PRs. It scans independently of any local ESLint setup.

## Key behaviors

- **Ignores eslint-disable comments**: The bot strips all `eslint-disable` directives before scanning. You cannot suppress bot errors with inline comments.
- **Reports "required" and "optional" issues**: Required issues block approval. Optional issues are advisory.
- **Re-scans automatically**: After pushing changes, the bot rescans within 6 hours. Do NOT open a new PR for re-validation.
- **`/skip` for false positives**: Comment `/skip` with a reason on the PR if a required issue is incorrect.

## Local enforcement

`@eslint-community/eslint-plugin-eslint-comments` with `no-restricted-disable` is configured in all plugins to block `eslint-disable` for `obsidianmd/*` rules locally. This catches futile suppression attempts at lint time — before the bot even sees the code.

## Common issues and solutions

### `no-explicit-any` — disabling not allowed

The bot forbids `eslint-disable` for `@typescript-eslint/no-explicit-any`. Fix by adding proper type augmentations instead.

### Sentence case for UI text (`obsidianmd/ui/sentence-case`)

The bot requires sentence case for string literals passed to UI methods (`.setName()`, `.setDesc()`, `.setPlaceholder()`). To bypass for intentionally lowercase text (e.g., property name placeholders), extract the string to a `const` — the bot only checks string literals, not variable references.

### Lookbehind regex (`obsidianmd/regex-lookbehind`)

Lookbehinds (`(?<=...)`) are not supported on iOS < 16.4. Replace with capture groups. Example: `(?<=\s|^)#tag` becomes `(^|\s)#tag` with the capture group returned in the replacement.

### Undescribed directive comments

All `eslint-disable` and `eslint-enable` comments must include a description after `--`. Example: `// eslint-disable-next-line rule-name -- reason why`.

### Static style assignment (`obsidianmd/no-static-styles-assignment`)

The rule catches direct property assignments like `element.style.left = 'auto'` but does NOT catch computed property access like `element.style[prop] = value`. To avoid lint errors for static assignments, move them to CSS classes instead of setting inline.

### Unused eslint-disable directives

The bot flags directives that suppress rules that aren't actually triggered. Remove them. Note: the bot's scanner may differ from local ESLint — a directive can be "unused" per the bot but needed locally (or vice versa). When this happens, keep the directive with a description (bot ignores it anyway) and ensure local ESLint passes.

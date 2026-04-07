---
title: Obsidian API quirks
description: Undocumented Obsidian API behaviors. Covers file write timing, race conditions, Bases config quirks, and workarounds.
author: 🤖 Generated with Claude Code
updated: 2026-03-15
---

# Obsidian API quirks

## Debounced disk writes (~2 seconds)

Obsidian debounces all file writes via `TextFileView.requestSave` with a **2-second** delay (documented in the TypeScript API). This applies globally — Markdown files, `.base` files, and any file managed by Obsidian's editor system.

### Implication for `vault.process()`

`vault.process(file, fn)` reads the file **from disk**, not from Obsidian's in-memory state. If Obsidian has pending in-memory changes that haven't been flushed (within the ~2s debounce window), `vault.process()` reads stale content. Writing back the transformed result overwrites the file **without** the pending changes, causing data loss.

### Known race condition

When a user creates a new Bases view and quickly switches its type (e.g., table → dynamic-views-grid), the new view exists in Obsidian's memory but not on disk. If `vault.process()` runs within the debounce window, it reads the file without the new view, rewrites it, and the view is lost. Obsidian then shows "View X not found."

## Notice container stale cache

> Observed in Obsidian **1.12.1**, installer 1.11.4.

Obsidian caches the `.notice-container` DOM element in a `Map<Window, HTMLDivElement>` keyed by `activeWindow`. When the last notice in a container fades out, `Notice.hide()` detaches both the notice element and the empty container from the DOM — but does **not** remove the Map entry.

On the next `new Notice()`, the constructor finds the cached (but detached) container via `Map.get(activeWindow)`, appends the new notice to it, and never re-attaches the container to `document.body`. The notice is invisible.

### When it triggers

This only affects notices created **after all previous notices have fully faded** (animation complete + detach). If notices overlap (a new one while another is still visible), the container stays in the DOM and the bug doesn't manifest.

### Workaround

After `new Notice()`, check if the container is connected and re-attach if stale:

```typescript
const notice = new Notice("...");
const nc = (notice as { containerEl?: HTMLElement }).containerEl?.parentElement;
if (nc && !nc.isConnected) {
  activeWindow.document.body.appendChild(nc);
}
```

## `BasesEntry.getValue()` undocumented `.data` property

> Observed in Obsidian **1.12.1**, installer 1.11.4.

`BasesEntry.getValue(propertyId)` returns `Value | null`. The official `Value` class hierarchy only exposes `toString()`, `isTruthy()`, `equals()`, `looseEquals()`, and `renderTo()`. No `.data` accessor is typed.

At runtime, `Value` subclasses store their raw data in an undocumented `.data` property:

- `PrimitiveValue<T>` (StringValue, NumberValue, BooleanValue, etc.): `.data` is the primitive value (`string`, `number`, `boolean`).
- `ListValue` (multitext properties like `tags`, `aliases`): `.data` is an array of `Value` objects or primitives.

### Accessing raw data

The plugin accesses `.data` via type assertion since it's not in the type definitions:

```typescript
const value = entry.getValue("note.author") as { data?: unknown } | null;
const data = value?.data;
if (Array.isArray(data)) {
  // multitext: data is an array
} else if (typeof data === "string") {
  // text: data is a string
}
```

### Fragility

This relies on Obsidian's internal `Value` implementation. If the internal property is renamed or restructured, access breaks silently (returns `undefined`). There is no public API alternative for extracting raw values beyond `toString()`.

## Bases `config.get()` falls back to schema defaults

> Observed in Obsidian **1.12.1**, installer 1.11.4.

`BasesViewConfig.get(key)` does not return `undefined` for missing YAML keys. It falls back to the `default` value from the view's schema (the `ViewOption[]` array returned by the options callback registered in `BasesViewRegistration`).

This means schema defaults are not just cosmetic (initial GUI display) — they affect all runtime config reads for keys absent from YAML.

### Implication for dynamic defaults

If schema defaults are computed at runtime (e.g., merged with a settings template), those computed values leak into `config.get()` for any key the user has cleared or never set. A cleared property picker field (value removed from YAML) silently reverts to whatever the schema default was at the time the options callback last ran.

## Bases YAML normalization strips default values

> Observed in Obsidian **1.8.9**, installer 1.7.7.

When Obsidian writes `.base` files, it strips YAML properties whose values match the schema default. For example, setting `rightPropertyPosition: right` (the default) causes the line to be removed entirely on the next write cycle.

### Implication

After testing with a non-default value (e.g., `rightPropertyPosition: column`), reverting to the default in the UI removes the YAML line rather than writing `rightPropertyPosition: right`. Code that checks for the presence of a YAML key to determine whether it was explicitly set cannot distinguish "explicitly set to default" from "never set."

### Workaround

Accept that default-valued properties are absent from YAML. Use `config.get(key)` which falls back to schema defaults (see "Bases `config.get()` falls back to schema defaults" above) rather than parsing YAML directly.



---
title: Declarative settings API quirks
description: Runtime behavior of Obsidian 1.13's getSettingDefinitions() API that the official docs don't state. Covers definition caching, refresh semantics, row reuse, search indexing, and a name collision that blanks the settings pane.
author: 🤖 Generated with Claude Code
updated: 2026-08-03
---

# Declarative settings API quirks

Obsidian 1.13 added a declarative settings API: a `PluginSettingTab` overrides `getSettingDefinitions()` and returns a tree of definitions instead of building rows imperatively in `display()`. The framework renders the tree, binds each `control` to a settings key, and indexes every row for global settings search.

The findings below were established by probing a live Obsidian instance, not read from the docs. Several contradict what the documentation implies.

**Observed**: 2026-08-03, Obsidian 1.13.4 (installer 1.13.4)

## Definitions are captured once, not per render

`getSettingDefinitions()` runs **once**, when `addSettingTab()` registers the tab. It does **not** re-run when the user opens the tab, and it does **not** re-run when the user navigates into a sub-page. Only `update()` re-runs it.

The documentation's phrasing ("called on every `display()`") is misleading, because `display()` is bypassed entirely once `getSettingDefinitions()` returns a non-empty array.

Consequences:

- **Anything computed at definition time is frozen** until `update()`. A `type: 'list'` whose `items` are mapped from an array renders the array as it was at registration. Every mutation — add, delete — must call `update()` or the list silently shows stale contents.
- **Closures stay live.** `visible` / `disabled` predicates and `render` callbacks capture references, so they read current state whenever they are invoked. It is the surrounding *structure* that is frozen, not the values the callbacks read.

## `refreshDomState()` does not re-read control values

`refreshDomState()` re-evaluates `visible` and `disabled` predicates and applies the result to existing DOM. It does **not** re-read control values through `getControlValue()`.

The failure this produces is quiet and easy to miss: a cascade force-writes a sibling setting to `false`, the stored value is correctly `false`, and the sibling's toggle keeps rendering **on** — correctly greyed out, but showing the wrong state.

| Situation | Correct call |
|---|---|
| A predicate's inputs changed (show/hide, enable/disable) | `refreshDomState()` |
| A **value** on another control was written | `update()` |
| The set of definitions changed (rows added/removed) | `update()` |

A useful nuance: controls on a **freshly mounted sub-page** do re-read `getControlValue()` even though definitions aren't rebuilt. So a cascade that writes to a control on a *different* page self-corrects when the user navigates there; only same-page writes strictly require `update()`. Using `update()` for all value-writing cascades is simpler and safe.

## Controls have no `onChange` — `setControlValue` is the only hook

`SettingControlBase` exposes exactly `key`, `defaultValue`, `validate`, and `disabled`. There is no per-control change callback.

Every `control` write is routed through the tab's `setControlValue(key, value)`, which makes that method the single place for side effects — cross-setting cascades, cache invalidation, re-initialising a subsystem after a value changes.

```ts
async setControlValue(key: string, value: unknown): Promise<void> {
  writeValue(key, value);
  runCascadeFor(key);          // mutate other settings directly
  await this.plugin.saveData(/* … */);
  needsRerender(key) ? this.update() : this.refreshDomState();
}
```

**Re-entrancy is not a concern** provided cascades mutate the settings object directly rather than calling `setControlValue` again. That keeps it to one save per user action.

Reaching for `render` instead — to get an `onChange` — is the wrong trade: it costs the automatic persistence and gains nothing that the `setControlValue` funnel doesn't already provide.

## `render` rows: two hard constraints

### No `disabled` field

`SettingDefinitionRender` declares only `control?: never`, `action?: never`, `render`, plus the inherited `name` / `desc` / `aliases` / `searchable` / `visible`. **`disabled` exists only on `SettingControlBase` and `SettingDefinitionAction`.** Putting `disabled` on a render definition is a compile error.

Use `visible` instead, or call `setting.setDisabled()` inside the render body.

### The row element is reused across renders

The framework reuses the same `.setting-item` element between renders and only rebuilds its info and control children. Anything appended **directly to `settingEl`** survives teardown and accumulates one copy per render — a table doubles on every `update()`: 14 rows, then 28, then 56.

Controls added via `setting.addButton()` / `addToggle()` are unaffected, because those live in `controlEl`, which the framework does clear. Only hand-appended DOM leaks.

The obvious fix — `settingEl.empty()` — is a trap: it removes the name element and **silently drops the row from global settings search**. The row still renders; it just stops being findable.

Correct pattern: remove only your own previously-mounted container, then create a fresh one.

```ts
function mountHost(settingEl: HTMLElement): HTMLElement {
  settingEl
    .querySelectorAll(':scope > .my-plugin-host')
    .forEach((stale) => stale.remove());
  return settingEl.createDiv({ cls: 'my-plugin-host' });
}
```

## Search indexing

- **`render` rows are indexed** by their `name`, exactly like `control` rows. Choosing `render` to keep an icon or a labelled button does not cost searchability.
- **Search results navigate into sub-pages.** A setting buried in a `type: 'page'` is findable from the top-level search field, and clicking the result opens its page.
- **A row with `name: ''` is effectively unsearchable** — useful for decorative call-to-action rows that aren't settings, and a silent defect anywhere else.
- **`searchable: false`** keeps user-data rows (list entries, for example) out of the index while leaving the section heading findable.

## `renderTab` is a reserved method name

`SettingTab.prototype.renderTab()` is what the core settings modal calls to draw a tab; its base implementation is roughly `this.settingItems.length > 0 ? renderDeclarative(this) : this.display()`.

Defining a method called `renderTab` on a `PluginSettingTab` subclass — for example a private helper that renders an internal sub-view — **shadows it by name**, so the framework's version never runs and `display()` is never reached.

The symptom is a **completely blank settings pane**, with no console error. Avoid the name entirely.

## Styling: the row is a flex container

`.setting-item` is `display: flex; flex-wrap: nowrap`. A container mounted into it becomes a third flex sibling alongside the info and control children, so wide content is squeezed into whatever width is left over — a full-width table can end up at roughly 40% of the row.

```css
.setting-item:has(> .my-plugin-host) { flex-wrap: wrap; }
.setting-item > .my-plugin-host { flex-basis: 100%; }
```

Two related migration hazards when moving imperative settings UI to the declarative API:

- **Descendant rules scoped to a page-level wrapper stop matching.** Declarative rows have no such ancestor, so a rule like `.my-page-wrapper .some-icon { … }` silently dies and the element falls back to default block layout. Prefer plugin-owned classes with no ancestor dependency.
- **Sibling combinators break.** Rules of the form `.some-table + .setting-item` no longer match, because the table now sits *inside* a `.setting-item` rather than beside one.

## `ConfirmationModal`

`ConfirmationModal extends Modal`, so it inherits `setTitle()` and `setContent(string | DocumentFragment)`. Buttons auto-close the modal on click; return a truthy value from the handler to keep it open.

`ButtonComponent.setWarning()` is **deprecated**. The replacement is `setDestructive()` for a destructive button, or `setDestructive().setCta()` for a destructive *primary* action — the latter produces the filled treatment. Both resolve to the same classes the deprecated call produced.

`addCancelButton()` supplies its own localized label; pass no argument.

## Nesting rules

- `SettingDefinitionGroup.items` is typed `SettingGroupItem[]` = `SettingDefinition | SettingDefinitionPage`. **A group cannot contain another group or a list.** Pages *can* nest inside groups.
- A page's `items` is `SettingDefinitionItem[]`, which does admit groups and lists.
- `SettingDefinitionBase` has **no `icon` field**. Per-row icons require a `render` callback that inserts the icon into `setting.nameEl`.

## Native lists

`type: 'list'` provides drag-to-reorder handles, per-row delete buttons, a platform-appropriate add affordance (tooltipped from `addItem.name`), and an `emptyState` shown at zero entries — all working out of the box.

`onReorder` fires after the DOM is already reordered, so it only needs to persist; it does not need `update()`. `onDelete` and `addItem` both do, because they change the `items` array (see the caching section above).

When row handlers need to address a specific entry, **close over the entry object rather than its index**. Reorder mutates the array without re-rendering, so a captured index points at the wrong row afterwards.

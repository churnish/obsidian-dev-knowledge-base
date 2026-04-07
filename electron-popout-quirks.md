---
title: Electron popout window quirks
description: Platform-specific quirks when running plugin code in Electron popout (BrowserWindow) windows.
author: 🤖 Generated with Claude Code
last updated: 2026-03-15
---

# Electron popout window quirks

Obsidian's "Open in new window" creates an Electron `BrowserWindow` (popout) with a separate V8 isolate and separate document. Plugin JS runs in the main window's context but operates on DOM elements in the popout's document. This causes several non-obvious issues.

## Module-scope `document`/`window` resolve to main window

All bare references to `document`, `window`, `requestAnimationFrame`, `ResizeObserver`, etc. at module scope resolve to the main window. For popout-aware code, derive from the element:

```ts
const doc = el.ownerDocument;
const win = doc.defaultView ?? window;
win.requestAnimationFrame(() => { ... });
new win.ResizeObserver((entries) => { ... });
```

**Affected APIs**: `document.body`, `document.activeElement`, `document.hasFocus()`, `document.createElement()`, `window.innerWidth/innerHeight`, `window.focus()`, `ResizeObserver`, `IntersectionObserver`, `requestAnimationFrame`, `addEventListener` on `document`/`window`.

## Cross-context observers silently fail

`ResizeObserver` and `IntersectionObserver` created in the main window's JS context silently fail to observe elements in a popout's DOM. The constructor must come from the popout's window:

```ts
// Wrong: uses main window's RO constructor
new ResizeObserver(callback).observe(popoutElement);

// Right: uses popout's RO constructor
const win = popoutElement.ownerDocument.defaultView ?? window;
new win.ResizeObserver(callback).observe(popoutElement);
```

Diagnostic: `ownerDocument.defaultView.ResizeObserver !== window.ResizeObserver` → `true` in popouts.

## `requestAnimationFrame` IDs are per-window

`cancelAnimationFrame(id)` must be called on the same window where `requestAnimationFrame(cb)` was called — RAF IDs are scoped to their originating V8 isolate. Canceling an ID on a different window is a silent no-op.

When tearing down state for a popout move, cancel all pending RAFs BEFORE nullifying the stored window reference. Otherwise, the `?? window` fallback targets the main window and the cancel does nothing — the orphaned callback fires in the old window's context.

```ts
// Wrong: observerWindow already null, cancel targets main window (no-op)
this.observerWindow = null;
(this.observerWindow ?? window).cancelAnimationFrame(this.rafId);

// Right: cancel while observerWindow still points to the popout
(this.observerWindow ?? window).cancelAnimationFrame(this.rafId);
this.rafId = null;
this.observerWindow = null;
```

## renderHash must be invalidated on document change

When a view moves between windows, `handleDocumentChange` tears down observers, but the underlying data hasn't changed. If the render pipeline uses a hash-based early return to skip redundant re-renders, the hash must be invalidated — otherwise the pipeline hits the early return, skipping observer re-creation in the new window context. CSS Grid views survive because they auto-reflow without JS; absolutely-positioned views (masonry) require explicit observer-driven layout.

```ts
// In handleDocumentChange:
this.teardownObservers();
this.renderState.lastRenderHash = ''; // Force pipeline to fall through
```

## No re-hit-test after overlay removal

After removing a DOM overlay (`cloneEl.remove()`), Electron popout windows do **not** recalculate `:hover` or dispatch `mouseenter`/`mouseleave` on elements that were underneath. In the main window, `:hover` updates after a `requestAnimationFrame`; in popouts, it stays stale indefinitely.

**Impact**: Any code that checks `element.matches(":hover")` or waits for `mouseenter` after removing an overlaying element will get incorrect results in popouts.

**Workaround**: Apply state changes (e.g., class additions) directly rather than depending on browser re-hit-testing. See `closeImageViewer()` in `image-viewer.ts` for an example.

## Panzoom `isAttached` check

The `@panzoom/panzoom` library's `isAttached` check walks up the DOM to find `document` (module scope). In popouts, the element is in a different document, so the check fails. Workaround: temporarily reparent the container to `document.body` during init, then move it back. See `setupImageViewerGestures()` in `image-viewer.ts`.

## Event listener binding

Libraries that bind event listeners to module-scope `document` (e.g., `pointermove`, `pointerup` for drag handling) will miss events in popouts since pointer events fire on the popout's document. Must rebind to the popout's document after init. See panzoom rebinding in `image-viewer.ts`.

## `defaultView` is null after window close

When a popout's `BrowserWindow` is closed, `ownerDocument.defaultView` returns `null` for elements that were in that document. Cleanup code that derives the window via `containerEl.ownerDocument.defaultView` must null-check — otherwise a `?? fallback` pattern may trigger unintended global behavior (e.g., cleaning up all windows' observers instead of just the closed one).

```ts
// Wrong: falls through to global cleanup when window is gone
cleanupObserver(el.ownerDocument.defaultView ?? undefined);

// Right: skip cleanup — observer dies with its window
const win = el.ownerDocument.defaultView;
if (win) cleanupObserver(win);
```

## Style Settings classes in popout windows

Style Settings plugin syncs `class-toggle` and `class-select` settings to ALL open documents (main + popouts). Changes made after popout creation DO reflect in popout windows — Style Settings handles this internally.

However, module-scope `document.body` (main window) remains the canonical source for reading configuration classes, because:

- It's always available (popout may not exist yet)
- It avoids coupling to a specific popout's document lifecycle

```ts
// Right: reads from main window body (canonical source)
const isExtMode = document.body.classList.contains('dynamic-views-file-type-ext');

// Right: creates node in correct document context
const textNode = cardEl.ownerDocument.createTextNode(title);
```

**Observed**: 2026-03-12, Obsidian 1.8.9, Style Settings 1.0.9

## Obsidian recreates `.view-content` during popout move

When a leaf is moved to a popout window (right-click → "Open in new window"), Obsidian destroys and recreates the `.view-content` element. Any inline styles set on `.view-content` before the move are lost. The view instance itself survives — only the DOM subtree is rebuilt.

Additionally, Obsidian mirrors the main window's body classes to the popout's body at creation time. This means body-class-gated CSS rules apply immediately once stylesheets load, even before any plugin JS runs in the popout context.

**Observed**: 2026-03-12, Obsidian 1.8.9

## Workspace event timing in popouts

All workspace events (`window-open`, `layout-change`) fire at ~300-380ms after popout creation. The first paint happens before any JS event fires, so workspace events cannot prevent FOUC. Style injection must happen synchronously during popout creation (e.g., via `window-open` event on the main window's `workspace`) or through CSS-only solutions.

**Observed**: 2026-03-12, Obsidian 1.8.9

## ResizeObserver doesn't fire for minimized windows

Electron does not dispatch `ResizeObserver` callbacks for minimized `BrowserWindow` instances. The window must be restored/shown before testing RO-based behavior. This applies to both main and popout windows.

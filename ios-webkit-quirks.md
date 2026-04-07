---
title: iOS WebKit quirks
description: Platform-specific bugs in iOS WebKit (WKWebView) affecting content-visibility, IntersectionObserver, CSS scroll-state(), and compositor layer shifts.
author: 🤖 Generated with Claude Code
last updated: 2026-03-10
---

# iOS WebKit quirks

## content-visibility: hidden + IntersectionObserver loop

Toggling `content-visibility: hidden` via IntersectionObserver causes an infinite reflow loop on iOS WebKit (WKWebView). The cycle: IO callback toggles class → geometry changes → IO re-fires → class toggles back → repeat. Flickering is persistent and never stops, even after all user interaction ceases.

**Root cause**: iOS WebKit re-evaluates IntersectionObserver entries when `content-visibility: hidden` changes an observed element's geometry. Chromium does not.

**Fix**: Use `content-visibility: auto` (browser-managed) on mobile instead of IO-driven toggling. Guarded via `Platform.isMobile` in `src/shared/content-visibility.ts`.

**Observed**: 2026-02-11, iOS.

## content-visibility: auto + masonry measurement

`content-visibility: auto` causes iOS WebKit to return the `contain-intrinsic-height` fallback (e.g., 300px) from `offsetHeight` for off-screen cards, even after an initial accurate measurement. This breaks masonry layout — positions are calculated from wrong heights, producing large gaps between cards. Chromium returns accurate heights regardless.

**Root cause**: iOS WebKit reports intrinsic fallback height for off-screen elements with `content-visibility: auto` when read via `offsetHeight`/`getBoundingClientRect()`. A ResizeObserver feedback loop compounds the issue — height changes from `auto` kicking in trigger re-measurement with wrong values.

**Fix**: Add `.masonry-measuring` class (which forces `content-visibility: visible !important`) around height reads in both the full layout and incremental (infinite scroll) layout paths.

**Observed**: 2026-02-11, iOS.

## CSS scroll-state() container queries

`CSS.supports('container-type', 'scroll-state')` returns `false` on iOS WebKit (tested 2026-02-11, Obsidian early access via TestFlight). The `@container scroll-state(stuck: top)` rule is silently ignored — progressive enhancement, no errors.

## Opacity on `<a>` inside absolute-positioned elements

Changing `opacity` on an `<a>` element inside a `position: absolute` container causes iOS WKWebView to momentarily shift the container horizontally before snapping back. The shift is visible to users but undetectable by JavaScript (`getBoundingClientRect()` reports no change) — it occurs at the compositor layer.

**Trigger**: Obsidian's `a.mobile-tap { opacity: 0.5 }` rule (applied on touchstart, removed on touchend) on card title `<a>` elements inside masonry cards.

**Fix**: Override with `.card-title a.mobile-tap { opacity: 1; filter: opacity(0.5); }` — `filter: opacity()` is visually identical but uses a different compositor pipeline that doesn't trigger the shift. Do NOT add `transition: filter` — animating the filter also triggers the same compositor shift.

**Observed**: 2026-02-21, iOS.

## iOS text selection loupe blocks custom long-press interactions

iOS's native text selection loupe (magnifier) is driven by `UILongPressGestureRecognizer` at the UIKit layer — it operates BEFORE web events reach JavaScript. Any custom long-press interaction (hold-to-drag, long-press menus) on or near text content will trigger the loupe after ~300ms.

**What does NOT work**:
1. `selectstart` `preventDefault()` — the event never fires on iOS for touch selections.
2. `user-select: none` on parent containers — iOS checks the element directly under the finger, ignoring parent rules.
3. `getSelection().removeAllRanges()` on pointermove — loupe is already shown by the time JS runs.
4. `pointermove` `preventDefault()` in capture phase — pointer events are too late in the event chain.
5. Full-screen overlay with `pointer-events: auto` — blocks all input.
6. Overlay with `pointer-events: none` — iOS ignores it.

**Fix**: `touchmove` `preventDefault()` in **capture phase** during the interaction. Touch events operate closer to UIKit's native gesture recognizer layer than pointer events — this is the only mechanism that reaches the native layer to block the loupe.

```js
const blockTouchMove = (ev) => ev.preventDefault();
document.addEventListener('touchmove', blockTouchMove, {
  passive: false,
  capture: true,
});
// Remove on interaction end
```

**Observed**: 2026-03-09, iPadOS 18.

## iPadOS CSS cursor property

iPadOS does not support the CSS `cursor` property (except `text`). Apple locks cursor appearance at the system level — no web API workaround exists. Custom cursors (e.g., `cursor: col-resize` on drag handles) have no effect.

**Observed**: 2026-03-09, iPadOS 18.

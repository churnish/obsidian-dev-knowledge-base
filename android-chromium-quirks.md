---
title: Android Chromium quirks
description: Platform-specific quirks in Android Chromium WebView (Capacitor) affecting CSS environment variables, DOM timing, scroll behavior, and animation patterns.
author: "\U0001F916 Generated with Claude Code"
updated: 2026-03-26
---
# Android Chromium quirks

Platform-specific differences between Android Chromium WebView (Capacitor) and iOS WebKit (WKWebView) that affect Obsidian plugin development. Android uses `is-android` body class; detect via `Platform.isAndroidApp` or `document.body.classList.contains('is-android')`.

## CSS `env()` not available

`env(safe-area-inset-top)` resolves to `0px` on Android Capacitor WebView — CSS environment variables are not exposed. Obsidian works around this by setting `--safe-area-inset-top` (and bottom) as CSS custom properties on `body`.

**Cross-platform pattern**: `var(--safe-area-inset-top, env(safe-area-inset-top, 0px))` — custom property on Android, native env on iOS.

**Observed**: 2026-03-24, Obsidian 1.12.7, Pixel 8a (Android 16).

## DOM assembly delayed by FUSE

Android's FUSE filesystem abstraction makes directory listing 25-50x slower than direct access. DOM elements like `.mobile-navbar` may not exist when view constructors run, even though they exist by the time `onDataUpdated()` fires.

**Fix**: Lazy-init pattern — attempt in constructor, retry in first `onDataUpdated()` call.

**Observed**: 2026-03-24, Obsidian 1.12.7, Pixel 8a (Android 16).

## Scrollbar always visible

Chromium WebView renders CSS-styled scrollbars via `::-webkit-scrollbar` (body has `styled-scrollbars` class). Unlike iOS UIKit overlay scroll indicators, these do not auto-fade after scrolling stops.

**Implication**: Any technique that relies on waiting for scroll indicators to fade (e.g., delaying a `scrollTop` write until after indicator disappears) provides no benefit on Android.

## `scrollend` fires at true idle

Chromium's `scrollend` event fires 1ms after the last natural scroll event — at true scroll-idle, not at finger-lift. This differs from iOS where `scrollend` fires at finger-lift before momentum begins.

Fling deceleration is perfectly monotonic — no reverse-direction deltas at momentum end. iOS produces 5-50px of reverse-direction "deceleration bounce" over 1-5 frames.

**Observed**: 2026-03-24, Obsidian 1.12.7, Pixel 8a (Android 16).

## Text autosizing inflates rendered text beyond CSS font-size

Android Chromium WebView applies "font boosting" (TextAutosizer) that inflates text rendering beyond the CSS `font-size`. The boost factor varies by context (~1.15x observed). CSS relative units resolve against the **pre-boost specified size**, not the rendered size:

| Unit | Resolves to | Android example |
|---|---|---|
| `1em`, `1cap`, `1ex`, `1ic` | Pre-boost `SpecifiedFontSize()` | 14.99px |
| `getComputedStyle().fontSize` | Post-boost `ComputedSize()` | 17.24px |
| `1lh` | Post-boost `ComputedLineHeight()` | 22.40px |

**Impact**: SVG icons sized with `1em` appear smaller than adjacent boosted text, causing vertical misalignment.

**Fix**: Derive the boosted font-size from `1lh`:

```scss
width: calc(1lh / var(--dynamic-views-line-height-tight));
```

This equals `1em` on desktop/iOS (no boosting) and the boosted font-size on Android. The formula works because `1lh` is the only font-relative unit that reflects boosted metrics, and dividing by the line-height ratio recovers the font-size.

**Suppression mechanisms** (from Chromium source `text_autosizer.cc`):
- `text-size-adjust: none` — disables per-element (but also overrides a11y font scaling)
- `max-height` constraint on ancestor — tricks `BlockHeightConstrained()` check
- Flex items — `IsFlexItem()` check suppresses autosizing (tracked via `WebFeature::kTextAutoSizingDisabledOnFlexbox`)

**Source**: `css_to_length_conversion_data.cc` (unit resolution), `text_autosizer.cc` (boost algorithm). Relevant bugs: crbug.com/163359, crbug.com/779409.

**Observed**: 2026-03-25, Obsidian 1.12.7, Pixel 8a (Android 16).

### Offscreen elements are exempt from autosizing

Elements with `visibility: hidden` + `position: absolute` are outside normal text flow clusters and do NOT receive text autosizing. Measurement elements created offscreen will measure at un-boosted metrics, producing systematically undersized corrections when the correction is applied to boosted text.

**Fix**: Measure from actual rendered (in-flow, visible) elements instead of synthetic offscreen probes.

### Detecting the boost ratio via `width: 1em` probe

CSS `1em` resolves to `SpecifiedFontSize()` (pre-boost), while `getComputedStyle().fontSize` returns the boosted value. A probe element with `width: 1em` in the DOM gives the pre-boost font-size via `getBoundingClientRect().width`:

```typescript
const probe = doc.createElement('span');
probe.classList.add('my-probe'); // CSS: width: 1em; height: 0; position: absolute; visibility: hidden;
wrapper.appendChild(probe);
const preBoost = probe.getBoundingClientRect().width;
probe.remove();
const postBoost = parseFloat(getComputedStyle(wrapper).fontSize);
const boostRatio = postBoost / preBoost; // 1.0 on desktop, ~1.15+ on Android
```

The probe MUST be in the DOM (appended to an in-flow element) for `getBoundingClientRect` to return a non-zero width. The `position: absolute` + `visibility: hidden` on the probe is fine — the probe itself doesn't need to be autosized, only the inherited `1em` resolution matters.

**Observed**: 2026-03-25, Obsidian 1.12.7, Pixel 8a (Android 16).

## Double-rAF unnecessary

WebKit's passive scroll listener optimization collapses inline `style.setProperty()` transition + target into a single style recalculation when both are set in the same execution context. The double-rAF pattern (frame 1: set transition, frame 2: set target) forces two separate recalcs.

Chromium does not have this optimization — single rAF is sufficient for CSS transitions triggered via inline styles during scroll handlers.

**Observed**: 2026-03-24, Obsidian 1.12.7, Pixel 8a (Android 16).

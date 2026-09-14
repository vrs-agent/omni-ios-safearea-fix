# OmniRoute — iOS standalone safe-area fix (source patch)

**Target repo:** [`diegosouzapw/OmniRoute`](https://github.com/diegosouzapw/OmniRoute)  
**Target branch:** `release/v3.8.51` (or current stable release branch)  
**Issue:** Dashboard UI slides under the iOS status bar / clock when opened as a home-screen app (standalone PWA).

## Root cause

OmniRoute's PWA manifest and viewport config opt into full-bleed under the iOS status bar:

- `viewportFit: "cover"` tells Safari to draw the page edge-to-edge, including under the status bar.
- `statusBarStyle: "black-translucent"` keeps the status bar translucent with content visible behind it.
- `display: "standalone"` in `manifest.ts` enables home-screen app mode.

But there is **no iOS safe-area compensation** in the source. The only top-inset handling is the macOS Electron path (`--desktop-safe-top`, applied to `header` only when running as macOS Electron). On an iPhone home-screen app, the sticky `Header` and the mobile sidebar drawer draw from `y=0`, under the clock.

## Files to modify

All paths are relative to the OmniRoute repo root (`/app` in the container).

---

### 1. `src/app/globals.css`

**Location:** `src/app/globals.css`  
**Context:** Lines 31 (root vars) and 276–277 (`body.electron-macos` rule)

**Current state (lines 276–277):**
```css
body.electron-macos {
  --desktop-safe-top: 30px;
  --desktop-safe-bottom: 8px;
}
```

**Add immediately after line 277 (after the closing brace of `body.electron-macos`):**
```css
/* iOS standalone PWA safe-area — keeps dashboard below the status bar */
@media (display-mode: standalone) {
  @supports (padding-top: env(safe-area-inset-top)) {
    :root {
      --ios-safe-top: env(safe-area-inset-top);
      --ios-safe-bottom: env(safe-area-inset-bottom);
    }
  }
}
```

**Rationale:** Mirrors the Electron pattern but scoped to standalone PWA mode. Using CSS custom properties (`--ios-safe-top`) keeps it maintainable and composable with the existing `--desktop-safe-top` approach.

---

### 2. `src/shared/components/Header.tsx`

**Location:** `src/shared/components/Header.tsx`  
**Context:** Lines 208–210 (header element with sticky positioning)

**Current state (lines 208–210):**
```tsx
<header
  className="sticky top-0 z-10 flex items-center justify-between border-b border-black/5 bg-bg px-8 py-4 dark:border-white/5"
  style={{
    paddingTop: isMacElectron ? "calc(1rem + var(--desktop-safe-top))" : undefined,
  }}
>
```

**Replace line 210 with:**
```tsx
    paddingTop: isMacElectron
      ? "calc(1rem + var(--desktop-safe-top))"
      : "calc(1rem + var(--ios-safe-top, 0px))",
```

**Full context after change:**
```tsx
<header
  className="sticky top-0 z-10 flex items-center justify-between border-b border-black/5 bg-bg px-8 py-4 dark:border-white/5"
  style={{
    paddingTop: isMacElectron
      ? "calc(1rem + var(--desktop-safe-top))"
      : "calc(1rem + var(--ios-safe-top, 0px))",
  }}
>
```

**Rationale:** Extends the safe-area padding to iOS standalone mode. The `var(--ios-safe-top, 0px)` fallback ensures normal browsers (where `--ios-safe-top` is undefined) remain unaffected.

---

### 3. `src/shared/components/layouts/DashboardLayout.tsx`

**Location:** `src/shared/components/layouts/DashboardLayout.tsx`  
**Context:** Line 105 (mobile sidebar drawer)

**Current state (line 105):**
```tsx
className={`fixed inset-y-0 start-0 z-50 transform lg:hidden transition-transform duration-300 ease-in-out h-dvh overflow-y-auto ${
  sidebarOpen ? "translate-x-0" : "-translate-x-full"
}`}
```

**Add an inline style for iOS standalone top inset. Replace lines 104–106 with:**
```tsx
className={`fixed start-0 z-50 transform lg:hidden transition-transform duration-300 ease-in-out h-dvh overflow-y-auto ${
  sidebarOpen ? "translate-x-0" : "-translate-x-full"
}`}
style={{
  top: "var(--ios-safe-top, 0px)",
  bottom: "var(--ios-safe-bottom, 0px)",
}}
```

**Full context after change:**
```tsx
{/* Sidebar - Mobile: full viewport height with proper scroll containment */}
<div
  className={`fixed start-0 z-50 transform lg:hidden transition-transform duration-300 ease-in-out h-dvh overflow-y-auto ${
    sidebarOpen ? "translate-x-0" : "-translate-x-full"
  }`}
  style={{
    top: "var(--ios-safe-top, 0px)",
    bottom: "var(--ios-safe-bottom, 0px)",
  }}
>
  <Sidebar onClose={() => setSidebarOpen(false)} isMacElectron={isMacElectron} />
</div>
```

**Rationale:** The mobile sidebar drawer currently uses `inset-y-0` (top:0, bottom:0), which draws it under the status bar in standalone mode. Explicit `top`/`bottom` with the iOS safe-area vars clears the status bar and the home indicator.

**Note:** `inset-y-0` was removed from the className to avoid conflict with the inline `top`/`bottom` style.

---

## Optional enhancement: apply safe-area to the main layout root

If you want the entire dashboard content area (not just the header and mobile sidebar) to respect the safe area, add the following to `src/shared/components/layouts/DashboardLayout.tsx`:

**Location:** Line 82 (layout root div)

**Current state:**
```tsx
<div className="flex h-dvh min-h-0 w-full overflow-hidden">
```

**Add padding-top for iOS standalone:**
```tsx
<div
  className="flex h-dvh min-h-0 w-full overflow-hidden"
  style={{ paddingTop: "var(--ios-safe-top, 0px)" }}
>
```

**Rationale:** This ensures the entire dashboard content area clears the status bar, not just the header. However, the header fix (change #2) is usually sufficient since the header is the primary element that overlaps.

---

## Testing checklist

After applying the changes:

1. **Build the app:**
   ```bash
   npm run build
   ```

2. **Test in normal browser tab (desktop + mobile Safari):**
   - Header should have normal `py-4` padding (no extra top inset).
   - Mobile sidebar should extend full viewport height.
   - No regression in layout or scrolling.

3. **Test as standalone PWA (iPhone):**
   - Add to home screen: Safari → Share → Add to Home Screen.
   - Open from home screen (standalone mode).
   - Header should clear the status bar (clock visible, no overlap).
   - Mobile sidebar drawer should start below the status bar and end above the home indicator.
   - Content should not be obscured by the status bar or home indicator.

4. **Test on iPad (standalone mode):**
   - Same as iPhone testing.
   - Ensure landscape and portrait orientations both respect safe areas.

5. **Test on macOS Electron (regression check):**
   - Electron app should still apply `--desktop-safe-top: 30px` as before.
   - No change to Electron behavior.

---

## PR submission notes

- **Branch name:** `fix/ios-standalone-safe-area` (or similar)
- **PR title:** `fix: respect iOS safe-area insets in standalone PWA mode`
- **Labels:** `bug`, `ios`, `pwa`, `ui`
- **References:** Link to any related issues (e.g., "Fixes #XXXX — Dashboard overlaps iOS status bar in home-screen app mode")

**Commit message suggestion:**
```
fix: respect iOS safe-area insets in standalone PWA mode

- Add --ios-safe-top/bottom CSS vars scoped to @media (display-mode: standalone)
- Apply safe-area padding to dashboard Header (matching existing Electron pattern)
- Offset mobile sidebar drawer to clear status bar and home indicator

This prevents the dashboard UI from sliding under the iOS status bar when
opened as a home-screen app, while keeping normal browser tabs unaffected.
```

---

## Why this approach?

1. **Mirrors existing patterns:** The fix follows the same structure as the macOS Electron safe-area handling already in the codebase (`--desktop-safe-top`), making it consistent and maintainable.

2. **Scoped to standalone mode:** Using `@media (display-mode: standalone)` ensures normal browser tabs (desktop, mobile Safari in tab, etc.) are completely unaffected.

3. **Progressive enhancement:** The `@supports (padding-top: env(safe-area-inset-top))` wrapper ensures graceful fallback on browsers that don't support CSS env() or safe-area insets.

4. **No hardcoded values:** Using CSS custom properties (`--ios-safe-top`) instead of hardcoded pixel values keeps the fix flexible and composable.

5. **Minimal surface area:** Only three files changed, all in the UI layer. No backend, database, or API changes required.

---

**Scratch repo:** [`vrs-agent/omni-ios-safearea-fix`](https://github.com/vrs-agent/omni-ios-safearea-fix)  
**This doc:** `OMNIROUTE_SOURCE_PATCH.md` (documentation for porting the hot-patch to source)

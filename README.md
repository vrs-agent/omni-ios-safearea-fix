# OmniRoute dashboard — iOS standalone safe-area fix

**Temp scratch repo — documentation only, no application code.**

Records the exact hot-patch applied to the running `omniroute` container (`diegosouzapw/omniroute:latest`) to stop the dashboard UI from sliding up under the iOS status bar / clock when it is opened as a home-screen app (standalone PWA).

## Root cause

The app runs as a standalone PWA and explicitly opts into full-bleed under the iOS status bar:

- `viewportFit: "cover"` makes Safari draw the page under the status bar.
- `statusBarStyle: "black-translucent"` keeps that full-bleed (content visible behind a translucent bar).
- `display: "standalone"` in the manifest enables home-screen app mode.

But the app has **no iOS safe-area compensation** — the only top-inset handling is the macOS Electron path (`--desktop-safe-top`, applied to `header` only when running as macOS Electron). On an iPhone home-screen app the sticky `Header` and the sidebar draw from `y=0`, under the clock.

## Files & line numbers (source)

| File (relative to repo root `/app`) | Line(s) | What |
|---|---|---|
| `src/app/layout.tsx` | 20–21 | `viewport` export: `themeColor`, `viewportFit: "cover"` |
| `src/app/layout.tsx` | 35–43 | `appleWebApp`: `capable`, `title`, `statusBarStyle: "black-translucent"` |
| `src/app/manifest.ts` | 8 | `start_url: "/dashboard"` |
| `src/app/manifest.ts` | 10 | `display: "standalone"` |
| `src/shared/components/Header.tsx` | 208 | `header` element: `className="sticky top-0 ... px-8 py-4 ..."` |
| `src/shared/components/Header.tsx` | 210 | `paddingTop: isMacElectron ? "calc(1rem + var(--desktop-safe-top))" : undefined` — Electron-only |
| `src/shared/components/layouts/DashboardLayout.tsx` | 82 | layout root `h-dvh` |
| `src/shared/components/layouts/DashboardLayout.tsx` | 105 | mobile sidebar `fixed inset-y-0 start-0 ... h-dvh` |
| `src/app/globals.css` | 31 | `:root` `--desktop-safe-top: 0px` |
| `src/app/globals.css` | 276–277 | `body.electron-macos { --desktop-safe-top: 30px; --desktop-safe-bottom: 8px; }` |

Reference for the copied-to/source patch (upstream repo `diegosouzapw/OmniRoute`, so line numbers are what was running).

## The hot-patch applied (compiled artifact)

The container runs a **precompiled Next.js standalone build**. Editing source `tsx`/`css` does nothing to the running app unless you also rebuild. The live patch therefore targeted the **compiled app CSS** (hashed Next chunk) served to browsers:

- **File:** `/app/.build/next/static/chunks/2vzastg94m6fm.css`
- **Backup created:** `/app/.build/next/static/chunks/2vzastg94m6fm.css.bak-ios-safearea`
- **Insertion point:** immediately after the compiled `body.electron-macos{--desktop-safe-top:30px;--desktop-safe-bottom:8px}` rule.

### Injected block (verbatim)

```css
/* iOS standalone safe-area (VPS hot-patch: keep content below the status bar) */
@media (display-mode: standalone){
  @supports (padding-top: env(safe-area-inset-top)){
    header.sticky.top-0{padding-top:calc(env(safe-area-inset-top) + 1rem)}
    .fixed.inset-y-0.start-0{top:env(safe-area-inset-top)}
  }
}
```

**Effect:** only when opened as an installed/standalone home-screen app (`@media (display-mode: standalone)`) and only on browsers that expose `env(safe-area-inset-*)` (iOS Safari), it adds the status-bar inset to the sticky dashboard `Header` and offsets the mobile sidebar drawer. Browser normal-tab and desktop/non-powered layouts are untouched.

## Verification

Served and verified both routes:

- Direct (omniroute container, HTTP `localhost:20128`): `GET /_next/static/chunks/2vzastg94m6fm.css` → `200`, 340094 bytes, `safe-area-inset-top` count **3**, injected `@media (display-mode: standalone)` block present.
- Through nginx/`https://server.varunrs.in`: same asset → `200`, same bytes/count, block present.

## Port to a proper build (recommended upstream)

This is a hot-patch to a scratch doc for reference. To land it properly in the OmniRoute codebase (so it survives an image rebuild), make the equivalent source change instead of the compiled-CSS hacks:

1. In `src/app/globals.css`, next to the `body.electron-macos` rule (line 276), add the same `@media (display-mode: standalone)` / `@supports (padding-top: env(safe-area-inset-top))` block shown above (optionally route it through a `--ios-safe-top` custom property mirroring `--desktop-safe-top`).
2. Optionally, in `Header.tsx` line 210, also account for `env(safe-area-inset-top)` in the `paddingTop` (currently Electron-only `--desktop-safe-top`).
3. Rebuild and ship via the normal image/PR path.

> Scratch repo created by `vrs-agent` on request to record the exact change with file names and line numbers. Non-code; safe to delete once ported.
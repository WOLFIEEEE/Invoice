# ADA Sprint Incident Report

## Concorde Career Colleges Website (`concorde-www`)

**Date of Report:** March 5, 2026
**Incident Detected By:** Dan Vilione (Lead Developer)
**Incident Reported:** March 3, 2026
**Incident Resolved:** March 2, 2026 (commit `a811fb231`)
**Severity:** High (production UX regressions on desktop and mobile)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Context](#2-project-context)
3. [Theme Architecture and Build Pipeline](#3-theme-architecture-and-build-pipeline)
4. [The ADA Sprint: Full Commit Timeline](#4-the-ada-sprint-full-commit-timeline)
5. [What Was Introduced: The `custom` Library](#5-what-was-introduced-the-custom-library)
6. [Bug #1: Primary Menu Hover/Click State Broken](#6-bug-1-primary-menu-hoverclick-state-broken)
7. [Bug #2: All Sticky Footer CTAs Visible on Mobile](#7-bug-2-all-sticky-footer-ctas-visible-on-mobile)
8. [Bug #3: Horizontal Scrollbar on Mobile](#8-bug-3-horizontal-scrollbar-on-mobile)
9. [Additional Code Quality Issues in a11y.js](#9-additional-code-quality-issues-in-a11yjs)
10. [How the Issue Was Resolved](#10-how-the-issue-was-resolved)
11. [What Was Kept vs. What Was Disabled](#11-what-was-kept-vs-what-was-disabled)
12. [Build Process Violation: Detailed Explanation](#12-build-process-violation-detailed-explanation)
13. [Line-by-Line Analysis of a11y.css](#13-line-by-line-analysis-of-a11ycss)
14. [Line-by-Line Analysis of a11y.js](#14-line-by-line-analysis-of-a11yjs)
15. [Remediation Plan: Re-implementing the Valid Fixes](#15-remediation-plan-re-implementing-the-valid-fixes)
16. [Process Improvements Going Forward](#16-process-improvements-going-forward)
17. [Appendix A: File Paths Reference](#appendix-a-file-paths-reference)
18. [Appendix B: Git Commit Details](#appendix-b-git-commit-details)

---

## 1. Executive Summary

During the ADA (Americans with Disabilities Act) accessibility sprint conducted between February 23-27, 2026, a contractor from Pivotal Accessibility introduced a CSS/JS library into the `concorde_bs` Drupal theme that caused three production-impacting UX regressions:

1. **Desktop:** The primary navigation mega-dropdown menu (e.g., "Admissions") had broken hover and click states, rendering the dropdown unusable.
2. **Mobile:** All sticky footer CTA links became visible simultaneously, instead of only the intended ones for each viewport.
3. **Mobile:** A horizontal scrollbar appeared across the site due to content overflow.

The root cause was a new `custom` library (`css/a11y.css` and `js/a11y.js`) that was:
- Placed in non-standard directories (`css/` and `js/`) instead of the theme's source directories (`src/scss/` and `src/js/`).
- Not processed through the theme's established build pipeline (Sass + Rollup + PostCSS).
- Loaded with aggressive `!important` CSS overrides that broke Bootstrap utility classes and layout behavior.
- Not tested on desktop or mobile before merging.

Dan Vilione resolved the issue on March 2, 2026 by commenting out the library in `concorde_bs.info.yml`. The rest of the ADA sprint changes (template-level ARIA attributes, landmark roles, breadcrumb labels) were retained and merged to production.

---

## 2. Project Context

### Platform

| Property | Value |
|----------|-------|
| CMS | Drupal 10 |
| Hosting | Pantheon |
| PHP Version | 8.2 |
| Active Theme | `concorde_bs` (Bootstrap 5 subtheme) |
| Base Theme | `bootstrap5` |
| Theme Path | `web/themes/custom/concorde_bs/` |
| Branch (ADA work) | `ada` |
| Production Branch | `master` |

### Repository Branching Model

- `master` is the production branch deployed to Pantheon.
- `ada` is the feature branch where the accessibility contractor committed all ADA-related changes.
- Merges from `ada` into `master` are done periodically as sprints are completed.
- Pantheon build artifact branches exist alongside these (e.g., "Build artifacts added by Pantheon").

---

## 3. Theme Architecture and Build Pipeline

Understanding the theme's intended architecture is critical to understanding what went wrong.

### Directory Structure (Expected)

```
web/themes/custom/concorde_bs/
├── src/                          <-- SOURCE files (what developers edit)
│   ├── scss/                     <-- SCSS source
│   │   ├── styles.scss           <-- Main entry point
│   │   ├── _variables.scss
│   │   ├── _bootstrap.scss
│   │   ├── _components.scss      <-- Imports all component partials
│   │   ├── _reboot.scss
│   │   ├── _fonts.scss
│   │   └── components/           <-- Component SCSS partials
│   │       ├── _navbar.scss
│   │       ├── _buttons.scss
│   │       ├── _offcanvas.scss
│   │       └── ... etc
│   └── js/                       <-- JS source
│       ├── scripts.js            <-- Main entry point
│       └── components/           <-- Component modules
│           ├── ada.js            <-- Properly integrated ADA JS
│           ├── dropdown-megamenu.js
│           ├── offcanvas-submenu.js
│           └── ... etc
│
├── build/                        <-- Build scripts (Node.js)
│   ├── config.js                 <-- Path configuration
│   ├── styles.js                 <-- Sass compilation + PostCSS
│   ├── scripts.js                <-- Rollup bundling + Babel + Terser
│   ├── icon-font.js              <-- Icon font generation
│   ├── vendor.js                 <-- Vendor asset copying
│   └── watch.js                  <-- File watcher
│
├── assets/                       <-- COMPILED output (build artifacts)
│   ├── css/styles.css            <-- Compiled from src/scss/
│   ├── js/scripts.min.js         <-- Bundled from src/js/
│   ├── vendor/                   <-- Copied vendor libraries
│   ├── icons/                    <-- Generated icon font
│   └── fonts/
│
├── dist/                         <-- Distribution build
├── templates/                    <-- Twig templates
├── concorde_bs.info.yml          <-- Theme definition + library loading
├── concorde_bs.libraries.yml     <-- Library definitions
└── package.json                  <-- Node.js dependencies + scripts
```

### Build Pipeline

The build process is defined in `package.json`:

| Command | What It Does |
|---------|--------------|
| `npm run build` | Runs styles + scripts + icon-font + vendor in parallel |
| `npm run styles` | Compiles SCSS from `src/scss/` → `assets/css/` (Sass → PostCSS with autoprefixer + cssnano) |
| `npm run scripts` | Bundles JS from `src/js/` → `assets/js/` (Rollup + Babel + Terser) |
| `npm run dev` | Builds everything + starts watch mode |
| `npm run vendor` | Copies vendor assets from `node_modules/` to `assets/vendor/` |

**Key configuration from `build/config.js`:**

| Config Key | Path | Purpose |
|-----------|------|---------|
| `path.scss` | `src/scss` | SCSS source directory |
| `path.src_js` | `src/js` | JavaScript source directory |
| `path.css` | `assets/css` | Compiled CSS output |
| `path.js` | `assets/js` | Compiled JS output |

The build pipeline enforces:
- **Sass compilation** with variables, mixins, and Bootstrap integration
- **Autoprefixing** for cross-browser compatibility
- **Minification** via cssnano (CSS) and Terser (JS)
- **Linting** via Stylelint (CSS) and ESLint (JS)
- **Bundling** via Rollup with tree-shaking and deduplication

### How Libraries Are Registered in Drupal

Libraries are defined in `concorde_bs.libraries.yml` and attached globally via `concorde_bs.info.yml`.

**The `global-styling` library (the correct pattern):**

```yaml
global-styling:
  css:
    theme:
      assets/vendor/swiper/swiper-bundle.min.css: { minified: true }
      assets/icons/concorde-icons.min.css: { minified: true }
      assets/css/styles.css: {}                    # <-- Built from src/scss/
  js:
    assets/vendor/simplebar/dist/simplebar.min.js: { minified: true, attributes: { defer: true } }
    assets/vendor/swiper/swiper-bundle.min.js: { minified: true, attributes: { defer: true } }
    assets/js/scripts.min.js: { minified: true, attributes: { defer: true } }  # <-- Built from src/js/
```

All paths point to `assets/` — the build output directory. Source files in `src/` are never referenced directly.

### Properly Integrated ADA JavaScript (for comparison)

The file `src/js/components/ada.js` demonstrates the correct approach. It:

1. Lives in `src/js/components/` where all JS modules belong.
2. Uses ES module `export default` syntax compatible with the Rollup bundler.
3. Is imported in `src/js/scripts.js` via `import "./components/ada"`.
4. Gets compiled and bundled into `assets/js/scripts.min.js` automatically.
5. Requires no separate library definition — it's part of the existing `global-styling` library.

---

## 4. The ADA Sprint: Full Commit Timeline

All commits by `Prince <prince@pivotalaccessibility.com>` on the `ada` branch:

| # | Date | Commit | Message | Files Changed | Impact |
|---|------|--------|---------|---------------|--------|
| 1 | Feb 23 | `f7468803e` | Multiple roles announced for links and submenu | Templates | Safe - ARIA role improvements |
| 2 | Feb 23 | `85f14236d` | Fix 7 issues | Templates | Safe - Accessibility fixes |
| 3 | Feb 23 | `0eb174a17` | Fix breadcrumb issues and add js/css file | 7 files | **INTRODUCED** `css/a11y.css`, `js/a11y.js`, `custom` library |
| 4 | Feb 23 | `7b419c073` | fix reopen issues | Templates | Safe - Template fixes |
| 5 | Feb 23 | `7a19fcf9f` | breadcrumb state | Templates | Safe - Breadcrumb state |
| 6 | Feb 23 | `2d4645d40` | beadcrumb label change | Templates | Safe - Breadcrumb label |
| 7 | Feb 23 | `e606acfd1` | Provide sufficient label for breadcrumb | Templates | Safe - Breadcrumb ARIA |
| 8 | Feb 23 | `ab6c11d5d` | breadcrumb state for mobile | Templates | Safe - Mobile breadcrumb |
| 9 | Feb 23 | `0afe9be17` | submenu headings | 3 files | Mixed - Added `role='heading'` to submenus + tweaked `a11y.css` |
| 10 | Feb 27 | `1bee7b629` | Change focus indicator color and fix 9 issues | 2 files | **PROBLEMATIC** - Added overflow:auto to offcanvas, focus styles, jQuery dependency to `a11y.js` |
| 11 | Feb 27 | `f2fb99a39` | visible Schedule tour link | 1 file | **PROBLEMATIC** - Added `#sticky-footer a.d-none { display: inline-block!important }` |
| 12 | Feb 27 | `47d3aba1a` | overflow on sticky footer | 1 file | **PROBLEMATIC** - Added overflow:auto to sticky footer column |
| 13 | Feb 27 | `69f414f9d` | adjust with sticky footer | 1 file | **PROBLEMATIC** - Added forced width override on sticky footer |
| 14 | Feb 27 | `b04689437` | Resolve stash conflicts | Multiple | Stash conflict resolution |

**Key observation:** Commits 1-8 were safe template-level changes (ARIA attributes, landmark roles). The problems started with commit 3 (introducing the out-of-pipeline library) and escalated in commits 10-13 (adding increasingly aggressive CSS overrides).

---

## 5. What Was Introduced: The `custom` Library

### Commit `0eb174a17` (Feb 23) — Initial Introduction

This commit created three things:

**1. New file: `web/themes/custom/concorde_bs/css/a11y.css`**

A brand new `css/` directory was created at the theme root. This directory did not previously exist. The initial content was minimal:

```css
@media (max-width: 1200px) {
    .textchat_live_chat_widget_holder{
        height: 400px !important;
    }
}
```

**2. New file: `web/themes/custom/concorde_bs/js/a11y.js`**

A brand new `js/` directory was created at the theme root. This directory did not previously exist. The initial content was a skeleton Drupal behavior:

```javascript
console.log("Custom JS loaded");

(function (Drupal) {
  Drupal.behaviors.customBehavior = {
    attach: function (context, settings) {
      console.log("Drupal behavior works");
    }
  };
})(Drupal);
```

**3. Library definition added to `concorde_bs.libraries.yml`:**

```yaml
custom:
  version: 1.x
  css:
    theme:
      css/a11y.css: {}
  js:
    js/a11y.js: {}
  dependencies:
  - core/drupal
```

**4. Library enabled globally in `concorde_bs.info.yml`:**

The line `- concorde_bs/custom` was added to the global libraries list, meaning this CSS and JS would load on every single page of the website.

### How the Files Grew Over Subsequent Commits

The `a11y.css` file grew from 5 lines to 45 lines across 6 commits, each adding more CSS rules with escalating side effects.

The `a11y.js` file grew from 9 lines to 17 lines, adding jQuery dependency and DOM manipulation.

---

## 6. Bug #1: Primary Menu Hover/Click State Broken

### What Users Saw

When hovering over or clicking "Admissions" (or any primary nav item with a mega-dropdown), the dropdown either did not appear, appeared with broken styling, or the hover/active states did not display correctly. The screenshot in Dan's email shows the Admissions menu in a broken expanded state.

### The Offending CSS (from `a11y.css`, added in commit `1bee7b629`)

```css
.offcanvas.offcanvas-start.show#navbar-offcanvas{
    overflow: auto;
}
```

### How the Navigation System Works

The Concorde navigation uses a multi-layered architecture:

1. **Offcanvas container** (`#navbar-offcanvas`): On mobile, the navigation slides in from the left as a Bootstrap offcanvas component. On desktop (via the theme's `_navbar.scss`), it's rendered statically:

   ```scss
   .navbar-offcanvas {
     @include media-breakpoint-up(md) {
       --bs-offcanvas-width: 100%;
       position: static;
       height: auto;
       transform: none !important;
       visibility: visible;
     }
   }
   ```

2. **Mega-dropdown menus** (`.dropdown-megamenu`): On desktop, these are absolutely positioned dropdowns that appear on hover. The JavaScript in `dropdown-megamenu.js` listens for `mouseenter` events on `[data-bs-menu-show]` elements to toggle `.shown` class on `.dropdown-pane` elements.

3. **Offcanvas submenus** (`.offcanvas-submenu`): On mobile, these are nested offcanvas panels within the main offcanvas.

### Why `overflow: auto` Broke the Menu

Setting `overflow: auto` on `#navbar-offcanvas` creates a new **CSS stacking context** and **containing block** for positioned descendants. The mega-dropdown menus, which are absolutely positioned within the navbar, depend on the parent container NOT having `overflow` set, so they can visually overflow beyond its bounds.

When `overflow: auto` is applied:
- The dropdown menus become clipped to the boundaries of the offcanvas container.
- Mouse events may not register correctly because the overflow context changes how the browser calculates element positions and visibility.
- The dropdown backdrop (`.dropdown-backdrop`) and dropdown panes (`.dropdown-pane`) no longer render correctly relative to the page.

The selector `.offcanvas.offcanvas-start.show#navbar-offcanvas` technically targets the offcanvas when it has the `.show` class (i.e., when the mobile offcanvas is actively displayed). However, due to the theme's responsive overrides (making the offcanvas static on desktop), and depending on the state management of Bootstrap's offcanvas component, this rule can interfere with desktop rendering.

### What the Developer Was Trying to Achieve

The intent was likely to allow scrolling within the mobile navigation offcanvas when menu content overflows vertically. This is a valid accessibility concern — if the offcanvas content is taller than the viewport, users need to be able to scroll.

### How It Should Have Been Fixed

The overflow rule should have been scoped to mobile viewports only using a media query, and placed in the proper SCSS file:

```scss
// In src/scss/components/_offcanvas.scss
@include media-breakpoint-down(md) {
  .navbar-offcanvas.show {
    overflow-y: auto;
  }
}
```

This would allow scrolling in the mobile offcanvas without affecting the desktop mega-dropdown behavior.

---

## 7. Bug #2: All Sticky Footer CTAs Visible on Mobile

### What Users Saw

On mobile, the sticky footer (a fixed bar at the bottom of the viewport with call-to-action links) showed all CTA links simultaneously instead of only the ones intended for that viewport size. This created a cluttered, unusable interface.

### The Offending CSS (from `a11y.css`, added in commit `f2fb99a39`)

```css
#sticky-footer a.d-none{
    display: inline-block!important;
}
```

### How Bootstrap's `d-none` Works

Bootstrap's `d-none` utility class sets `display: none !important`. It is commonly used with responsive variants:

```html
<!-- This link is hidden by default, shown only on md+ -->
<a href="/tour" class="d-none d-md-inline-block">Schedule Tour</a>

<!-- This link is hidden by default, shown only on sm and below -->
<a href="/call" class="d-inline-block d-md-none">Call Now</a>
```

The Concorde sticky footer uses this pattern to show different CTAs at different viewport sizes. On mobile, some links are intentionally hidden with `d-none` while others are shown.

### Why This CSS Rule Broke Everything

The rule `#sticky-footer a.d-none { display: inline-block!important; }` targets **every single anchor tag** inside `#sticky-footer` that has the class `d-none`, and forces it to be visible. This is a blanket override that completely defeats the purpose of responsive visibility classes.

The result:
- Links intended only for desktop were now visible on mobile.
- Links intended only for mobile were now doubled up on all viewports.
- The sticky footer became overcrowded with all CTAs showing at once.

### What the Developer Was Trying to Achieve

Based on the commit message "visible Schedule tour link," the intent was to make a single specific link ("Schedule Tour") visible. The developer needed only one link to show up but wrote a selector that matched all hidden links in the sticky footer.

### How It Should Have Been Fixed

Instead of overriding all `d-none` links, the specific link should have been targeted — either by adding a custom class or by targeting the specific link text/URL:

```scss
// In src/scss/ within the appropriate component file
#sticky-footer {
  .schedule-tour-link {
    display: inline-block !important;
  }
}
```

Or better yet, the HTML markup in the Twig template should have been updated to remove `d-none` from that specific link and use proper responsive classes.

---

## 8. Bug #3: Horizontal Scrollbar on Mobile

### What Users Saw

On mobile viewports, a horizontal scrollbar appeared at the bottom of the page, allowing users to scroll left/right — indicating content was overflowing the viewport width.

### The Offending CSS (from `a11y.css`, added in commits `47d3aba1a` and `69f414f9d`)

```css
#sticky-footer .col-12.col-sm-10.col-md-8.mx-auto{
    overflow: auto;
}

@media (min-width: 768px) {
    #sticky-footer .col-md-8 {
        width: 82.666667% !important;
    }
}
```

### Why This Caused a Horizontal Scrollbar

**Problem 1: `overflow: auto` on the sticky footer column**

Setting `overflow: auto` on a full-width column (`.col-12`) inside the sticky footer means that if any child element exceeds the column width (even by 1px due to padding, borders, or margins), the browser will render a scrollbar. On mobile where `.col-12` means 100% width, any slight overflow becomes a horizontal scroll.

The sticky footer's CTA links, buttons, or other interactive elements inside this column can trigger this overflow if they have:
- Padding or margins that push them beyond the container.
- The forced `display: inline-block!important` from Bug #2 causing more elements to be laid out than expected.
- The `overflow: auto` itself allowing the overflow to scroll rather than being hidden or clipped.

**Problem 2: Forced width override**

The rule `#sticky-footer .col-md-8 { width: 82.666667% !important; }` overrides Bootstrap's grid column width with `!important`. Bootstrap's `.col-md-8` is already calculated to be `66.666667%` at the `md` breakpoint. Forcing it to `82.666667%` could push content beyond the container bounds when combined with margins and padding from `.mx-auto` and the grid gutter.

**Combined Effect:**

When Bug #2 made all CTAs visible AND Bug #3 added `overflow: auto` + width overrides, the sticky footer had too many visible elements in a container that was too wide, creating the horizontal scrollbar on mobile.

### How It Should Have Been Fixed

If the sticky footer needed more width at certain breakpoints, the proper approach would be to modify the Twig template markup to use wider Bootstrap grid classes (e.g., change `.col-md-8` to `.col-md-10`) or adjust the layout within the SCSS build pipeline using responsive mixins.

---

## 9. Additional Code Quality Issues in a11y.js

The `js/a11y.js` file had several problems beyond the UX bugs:

### Issue 1: Console.log Statements in Production

```javascript
console.log("Custom JS loaded");
// ...
console.log("Drupal behavior works");
```

These debug statements were left in the file and would execute on every page load for every visitor. They pollute the browser console and indicate the code was not prepared for production deployment.

### Issue 2: Undeclared jQuery Dependency

The library definition in `concorde_bs.libraries.yml` declares only `core/drupal` as a dependency:

```yaml
custom:
  dependencies:
  - core/drupal
```

However, `a11y.js` was changed in commit `1bee7b629` to use jQuery:

```javascript
(function ($, Drupal) {
  // ...uses $ (jQuery) throughout...
  $('.textchat-widget-bubble...', context).attr('aria-label','Chat Now');
  $('div[data-inline-block-uuid="..."] p a').removeAttr('tabindex');
})(jQuery, Drupal);
```

Without `core/jquery` in the dependencies, this code would throw a `ReferenceError: jQuery is not defined` on any page where jQuery wasn't already loaded by another library. It only worked by accident because other libraries on the page (like `data-layer` or `chatscript`) declare the jQuery dependency and happen to load before this script.

### Issue 3: Magic Timeout for Chat Widget

```javascript
setTimeout(()=>{
  $('.textchat-widget-bubble...', context).attr('aria-label','Chat Now');
}, 4000);
```

Using a hardcoded 4-second timeout to wait for the chat widget to render is fragile. If the chat widget loads slower (due to network conditions) or faster (due to caching), this code will either:
- Execute before the widget exists (doing nothing).
- Execute after a needless delay.

A more robust approach would be to use a `MutationObserver` or the chat widget's API callbacks.

### Issue 4: Hardcoded Block UUIDs

```javascript
$('div[data-inline-block-uuid="3b6f5d46-59a0-4b5a-8885-2ecc0b95ccc2"] p a').removeAttr('tabindex');
$('div[data-inline-block-uuid="db572fd9-e75b-4aca-822b-939c902f765a"] p a').removeAttr('tabindex');
```

These selectors target specific Drupal inline blocks by their UUID. This is extremely brittle because:
- If the blocks are ever deleted and recreated, they'll get new UUIDs.
- If the database is rebuilt (which Dan mentioned he did: "loaded a fresh copy of the production database"), UUIDs may differ.
- Other blocks that need the same fix won't benefit from this code.

A better approach would be to use a class-based or structural selector.

---

## 10. How the Issue Was Resolved

### Dan Vilione's Fix (Commit `a811fb231`, March 2, 2026)

Dan's commit message reads:

> Update composer dependencies, fix block configuration, and enhance JavaScript error handling
> - Disabled concorde_bs/custom to resolve frontend errors

The specific change in `concorde_bs.info.yml`:

```diff
-  - concorde_bs/custom
+#  - concorde_bs/custom
```

By commenting out the line `- concorde_bs/custom` from the theme's global library list, Drupal stops loading both `css/a11y.css` and `js/a11y.js` on any page. The files still exist in the repository, but they are inert.

### What Dan's Commit Also Included

The same commit addressed several other issues:
- Updated Composer dependencies (drupal/bootstrap5, drupal/consumers, drupal/helper, etc.)
- Fixed block configuration and module dependencies
- Added JavaScript error handling in `concorde_webform` module to handle undefined settings gracefully
- Updated distribution build assets (`dist/`)

### Post-Resolution: Branch Rebase and Database Refresh

From Dan's email:
> We also rebased master onto the ADA branch and loaded a fresh copy of the production database into ADA so your team can continue working from an updated baseline.

This means:
1. The `ada` branch now has all of `master`'s history, including the fix.
2. The ADA environment has a fresh production database, so any Drupal configuration changes from production are available.
3. The `custom` library is currently **disabled** on the `ada` branch.

---

## 11. What Was Kept vs. What Was Disabled

### Kept and Merged to Production

These changes from the ADA sprint were deemed safe and are now live:

| Change | Files Affected | Description |
|--------|---------------|-------------|
| ARIA heading roles on submenu titles | `header.html.twig` | Added `role='heading' aria-level='2'` to offcanvas submenu titles (Nursing Programs, Dental Programs, etc.) |
| Breadcrumb ARIA labels | `breadcrumb.html.twig` | Improved breadcrumb landmark labels |
| Breadcrumb mobile state | Templates | Breadcrumb responsive behavior |
| Footer ARIA improvements | `footer.html.twig` | Footer landmark and label improvements |
| Landmark roles for main content | Multiple templates | Added proper `<main>` landmark definitions |
| `src/js/components/ada.js` | Built JS (in pipeline) | Zoom-level detection for secondary nav and sticky footer |
| `src/js/components/main-content.js` | Built JS (in pipeline) | Main content landmark management |

### Disabled (Not in Production)

| Change | Files | Status |
|--------|-------|--------|
| `css/a11y.css` | `web/themes/custom/concorde_bs/css/a11y.css` | File exists but library not loaded |
| `js/a11y.js` | `web/themes/custom/concorde_bs/js/a11y.js` | File exists but library not loaded |
| `custom` library definition | `concorde_bs.libraries.yml` (lines 169-177) | Definition exists but not referenced |
| `custom` library loading | `concorde_bs.info.yml` (line 12) | Commented out: `#  - concorde_bs/custom` |

---

## 12. Build Process Violation: Detailed Explanation

### What the Contractor Did

```
web/themes/custom/concorde_bs/
├── css/              <-- NEW directory (should not exist)
│   └── a11y.css      <-- Raw CSS, not compiled
├── js/               <-- NEW directory (should not exist)
│   └── a11y.js       <-- Raw JS, not bundled
```

### What Should Have Been Done

```
web/themes/custom/concorde_bs/
├── src/
│   ├── scss/
│   │   └── components/
│   │       └── _a11y.scss      <-- New partial imported into _components.scss
│   └── js/
│       └── components/
│           └── a11y-behavior.js  <-- New module imported into scripts.js
```

Then run `npm run build` to compile into `assets/css/styles.css` and `assets/js/scripts.min.js`.

### Why This Matters

| Aspect | Build Pipeline | Raw Files (What Was Done) |
|--------|---------------|--------------------------|
| **Sass variables** | Has access to `$variables`, Bootstrap mixins, responsive breakpoints | Must hardcode pixel values and colors |
| **Autoprefixing** | Automatic vendor prefixes for cross-browser support | None — developer must manually add prefixes |
| **Minification** | Automatic via cssnano/Terser | None — raw unminified CSS/JS served to users |
| **Linting** | Stylelint + ESLint catch issues before build | No linting — code quality issues go undetected |
| **Bundling** | Single HTTP request for all JS | Additional HTTP request for each raw file |
| **Tree-shaking** | Unused code removed automatically | All code shipped regardless |
| **Consistency** | Uses shared Bootstrap breakpoints, colors, spacing | Hardcoded values that may drift from theme |
| **Cache busting** | Drupal aggregation handles versioning | May cause caching issues |

### The Resulting Technical Debt

The raw `css/a11y.css` file contains hardcoded values that should reference theme variables:

- `#5E5F68` — should reference a Sass variable for accessible placeholder color
- `300px`, `40px`, `82.666667%` — should use Bootstrap spacing/grid variables
- `@media (max-width: 1200px)` — should use Bootstrap's `@include media-breakpoint-down(xl)` mixin
- `@media (max-width: 667px)` — non-standard breakpoint, doesn't align with Bootstrap's breakpoint system
- `@media (min-width: 768px)` — should use `@include media-breakpoint-up(md)`

---

## 13. Line-by-Line Analysis of a11y.css

Below is every rule in the final version of `css/a11y.css` with its intent, problem assessment, and recommended action.

### Rule 1: Chat Widget Height (Lines 1-5)

```css
@media (max-width: 1200px) {
    .textchat_live_chat_widget_holder{
        height: 300px !important;
    }
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Reduce chat widget height on screens narrower than 1200px |
| Problem Level | Low risk |
| Issue | Uses `!important` to override inline styles; hardcoded breakpoint doesn't match Bootstrap's `xl` (1200px uses `max-width: 1199.98px` in Bootstrap) |
| Recommendation | Re-implement in `src/scss/` using `@include media-breakpoint-down(xl)`. Verify if the chat widget's own CSS can be configured instead. |

### Rule 2: Chat Widget Mobile Position (Lines 7-11)

```css
@media (max-width: 667px) {
    body .textchat_live_chat_widget_holder{
        bottom: 40px !important;
    }
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Move chat widget up by 40px on small screens so it doesn't overlap the sticky footer |
| Problem Level | Low risk |
| Issue | `667px` is a non-standard breakpoint (likely targeting iPhone landscape). Should use Bootstrap's breakpoint system. The `body` prefix is used for specificity boosting. |
| Recommendation | Re-implement in `src/scss/` using `@include media-breakpoint-down(sm)` or a custom variable. |

### Rule 3: Search Placeholder Color (Lines 13-15)

```css
#search-form input::placeholder{
    color: #5E5F68;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Improve contrast ratio of search form placeholder text for WCAG compliance |
| Problem Level | None — this is a safe, valid fix |
| Issue | Hardcoded color value instead of using a Sass variable |
| Recommendation | Re-implement in `src/scss/` with a Sass variable (e.g., `$input-placeholder-color-accessible`). |

### Rule 4: Focus Visible on White Links (Lines 17-19)

```css
body a.text-white.fw-bold:focus-visible,
body a.fw-semibold.text-white:focus-visible{
    outline: 5px auto -webkit-focus-ring-color;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Ensure keyboard focus indicators are visible on white text links (which are hard to see with default focus outlines on dark backgrounds) |
| Problem Level | None — this is a valid accessibility fix |
| Issue | Uses `-webkit-focus-ring-color` which is a non-standard WebKit property. The `body` prefix is used unnecessarily for specificity. |
| Recommendation | Re-implement in `src/scss/` using standard `outline-color` with a contrasting color variable. Consider using `:focus-visible` globally with proper theme colors. |

### Rule 5: Offcanvas Overflow (Lines 21-23) — BUG CAUSE

```css
.offcanvas.offcanvas-start.show#navbar-offcanvas{
    overflow: auto;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Allow scrolling in the mobile navigation offcanvas |
| Problem Level | **HIGH — caused Bug #1** |
| Issue | Not scoped to mobile. Creates new stacking context that breaks desktop mega-dropdown hover/click states. |
| Recommendation | If needed, scope to mobile only: `@include media-breakpoint-down(md) { .navbar-offcanvas.show { overflow-y: auto; } }` in `src/scss/components/_offcanvas.scss`. Test mega-dropdown on desktop after applying. |

### Rule 6: Chat Widget Input Border (Lines 25-27)

```css
#textchat_live_chat_widget .flex.flex-col.gap-2 .flex.flex-col.gap-1 input{
    border: 1px solid black;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Add visible border to chat widget form inputs for accessibility (inputs without borders are hard to identify for users with low vision) |
| Problem Level | None — valid fix |
| Issue | Very deep, fragile selector chain that depends on the chat widget's internal class structure. If the chat widget updates, this breaks. |
| Recommendation | Re-implement in `src/scss/` with a comment explaining the chat widget version this targets. Consider using a less fragile selector if possible. |

### Rule 7: Sticky Footer Hidden Links Override (Lines 29-31) — BUG CAUSE

```css
#sticky-footer a.d-none{
    display: inline-block!important;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Make the "Schedule Tour" link visible in the sticky footer |
| Problem Level | **HIGH — caused Bug #2** |
| Issue | Overrides `d-none` on ALL anchor tags in the sticky footer, not just the intended one. The `!important` defeats all responsive visibility classes. |
| Recommendation | Do not re-implement this rule. Instead, modify the Twig template to use correct responsive visibility classes on the specific link. If CSS is needed, target a specific class or data attribute on the intended link only. |

### Rule 8: Sticky Footer Column Overflow (Lines 33-35) — BUG CAUSE

```css
#sticky-footer .col-12.col-sm-10.col-md-8.mx-auto{
    overflow: auto;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Allow the sticky footer content to scroll if it overflows |
| Problem Level | **HIGH — contributed to Bug #3** |
| Issue | `overflow: auto` on a full-width column causes a horizontal scrollbar when combined with the forced visibility of all CTAs (Bug #2). |
| Recommendation | Do not re-implement. If content overflow is a concern, address the root cause (too many visible elements) rather than adding scrollbars to a footer CTA bar. |

### Rule 9: Sticky Footer Width Override (Lines 36-40) — BUG CAUSE

```css
@media (min-width: 768px) {
    #sticky-footer .col-md-8 {
        width: 82.666667% !important;
    }
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Widen the sticky footer column from Bootstrap's default `col-md-8` (66.67%) to ~83% |
| Problem Level | **MEDIUM — contributed to Bug #3** |
| Issue | Overrides Bootstrap's grid system with `!important`, breaking the column math. This could push content beyond the container width. |
| Recommendation | If a wider column is needed, modify the template markup to use `col-md-10` instead of overriding with CSS. |

### Rule 10: Tab Focus Border (Lines 42-45)

```css
#whyConcordTab .nav-link:focus-visible{
    border: 3px solid #484c51 !important;
    box-shadow: none !important;
}
```

| Aspect | Detail |
|--------|--------|
| Intent | Add visible focus indicator to "Why Concorde" tab navigation |
| Problem Level | Low risk |
| Issue | Hardcoded color and uses double `!important`. Tab name appears misspelled ("Concord" vs "Concorde"). |
| Recommendation | Re-implement in `src/scss/` using theme focus indicator variables. |

---

## 14. Line-by-Line Analysis of a11y.js

### Line 1: Debug Console Log

```javascript
console.log("Custom JS loaded");
```

**Problem:** Production code should never contain `console.log` statements. This executes on every page load for every user.

**Action:** Remove entirely.

### Lines 3-17: Drupal Behavior

```javascript
(function ($, Drupal) {
  Drupal.behaviors.customBehavior = {
    attach: function (context, settings) {
      console.log("Drupal behavior works");

      setTimeout(()=>{
        $('.textchat-widget-bubble.textchat-elements--right.textchat-elements--right.textchat-widget--expanded', context)
          .attr('aria-label','Chat Now');
      },4000);

      $('div[data-inline-block-uuid="3b6f5d46-59a0-4b5a-8885-2ecc0b95ccc2"] p a').removeAttr('tabindex');
      $('div[data-inline-block-uuid="db572fd9-e75b-4aca-822b-939c902f765a"] p a').removeAttr('tabindex');
    }
  };
})(jQuery, Drupal);
```

| Line | Issue |
|------|-------|
| `console.log("Drupal behavior works")` | Debug statement in production |
| `setTimeout(..., 4000)` | Fragile 4-second delay; chat widget may not be loaded yet or may already be loaded |
| `.textchat-elements--right.textchat-elements--right` | Duplicated class in selector (likely a copy-paste error) |
| `jQuery` (line 17) | Undeclared dependency — the library only declares `core/drupal`, not `core/jquery` |
| `data-inline-block-uuid="3b6f5d46..."` | Hardcoded UUID tied to a specific Drupal block instance |
| `data-inline-block-uuid="db572fd9..."` | Same problem — another hardcoded UUID |
| `Drupal.behaviors.customBehavior` | Generic name "customBehavior" provides no context about what it does |

### Recommended Re-implementation

If these behaviors are still needed, they should be implemented as a proper ES module in `src/js/components/`:

```javascript
// src/js/components/a11y-fixes.js
export default (() => {
  const observer = new MutationObserver((mutations) => {
    const chatBubble = document.querySelector('.textchat-widget-bubble.textchat-widget--expanded');
    if (chatBubble && !chatBubble.getAttribute('aria-label')) {
      chatBubble.setAttribute('aria-label', 'Chat Now');
    }
  });

  observer.observe(document.body, { childList: true, subtree: true });

  document.querySelectorAll('.block-inline-block a[tabindex]').forEach(link => {
    link.removeAttribute('tabindex');
  });
})();
```

Then import it in `src/js/scripts.js`:

```javascript
import "./components/a11y-fixes";
```

---

## 15. Remediation Plan: Re-implementing the Valid Fixes

### Step 1: Create a New SCSS Partial

Create `src/scss/components/_a11y.scss`:

```scss
// Accessibility fixes

// Chat widget adjustments
@include media-breakpoint-down(xl) {
  .textchat_live_chat_widget_holder {
    height: 300px !important;
  }
}

@include media-breakpoint-down(sm) {
  body .textchat_live_chat_widget_holder {
    bottom: 40px !important;
  }
}

// Search placeholder contrast (WCAG AA)
#search-form input::placeholder {
  color: $accessible-placeholder-color; // define in _variables.scss
}

// Focus indicators for white text links
a.text-white:focus-visible {
  outline: 3px solid $focus-ring-color; // define in _variables.scss
  outline-offset: 2px;
}

// Chat widget input border (accessibility)
#textchat_live_chat_widget input {
  border: $border-width solid $border-color;
}

// Why Concorde tab focus
#whyConcordTab .nav-link:focus-visible {
  border: 3px solid $gray-700;
  box-shadow: none;
}
```

### Step 2: Import in `_components.scss`

Add to `src/scss/_components.scss`:

```scss
@import "components/a11y";
```

### Step 3: Create New JS Module (If Needed)

Create `src/js/components/a11y-fixes.js` with MutationObserver-based chat widget fix and class-based selectors for tabindex removal.

### Step 4: Import in `scripts.js`

Add to `src/js/scripts.js`:

```javascript
import "./components/a11y-fixes";
```

### Step 5: Build

Run `npm run build` from the theme directory.

### Step 6: Test

Test on:
- Desktop: Chrome, Firefox, Safari, Edge
  - Verify all mega-dropdown menus open/close on hover and click
  - Verify keyboard navigation and focus indicators
  - Verify chat widget accessibility
- Mobile: iOS Safari, Android Chrome
  - Verify sticky footer shows correct CTAs per viewport
  - Verify no horizontal scrollbar exists
  - Verify offcanvas navigation scrolls properly
  - Verify chat widget does not overlap sticky footer

### Step 7: Clean Up

- Remove `web/themes/custom/concorde_bs/css/` directory and its contents
- Remove `web/themes/custom/concorde_bs/js/` directory and its contents
- Remove the `custom` library definition from `concorde_bs.libraries.yml` (lines 169-177)
- Remove the commented-out line from `concorde_bs.info.yml` (line 12)

---

## 16. Process Improvements Going Forward

### 1. Mandatory Cross-Device Testing

Every front-end pull request must include evidence of testing on:

- [ ] Desktop (1280px+) — navigation hover states, dropdown menus, focus indicators
- [ ] Tablet (768px-1024px) — responsive layout, offcanvas navigation
- [ ] Mobile (375px-480px) — sticky footer, no horizontal scrollbar, touch targets

Consider adding screenshots or screen recordings to PRs for visual changes.

### 2. Build Pipeline Enforcement

No raw CSS or JS files should be committed outside the `src/` directory. Enforcement options:

- **Git hook (pre-commit):** Reject commits that add/modify files in `web/themes/custom/concorde_bs/css/` or `web/themes/custom/concorde_bs/js/`.
- **Code review rule:** Any PR touching theme assets must show that `npm run build` was executed.
- **CI check:** Add a CI step that verifies no uncompiled CSS/JS exists outside `src/` and `assets/`.

### 3. Contractor Onboarding Documentation

Create a `CONTRIBUTING.md` in the theme directory that covers:

- Theme directory structure and the purpose of each folder
- Build pipeline commands (`npm run build`, `npm run dev`)
- Where to add new SCSS (in `src/scss/components/` as a partial)
- Where to add new JS (in `src/js/components/` as an ES module)
- How libraries are registered and loaded in Drupal
- Testing requirements before submitting code for review

### 4. Code Review Checklist for Accessibility Work

- [ ] No `console.log` statements in production code
- [ ] No `!important` overrides on Bootstrap utility classes without documented justification
- [ ] All CSS goes through the SCSS build pipeline
- [ ] All JS goes through the Rollup build pipeline
- [ ] jQuery dependency is declared if used
- [ ] No hardcoded UUIDs or magic numbers
- [ ] Responsive behavior tested at all breakpoints
- [ ] Focus indicators tested with keyboard navigation
- [ ] Screen reader testing performed (VoiceOver on Mac, NVDA on Windows)

### 5. Branching and Merge Strategy

- The `ada` branch should be regularly rebased or merged from `master` to prevent large divergences.
- Feature branches for specific issues (e.g., `ada/sticky-footer-fix`, `ada/focus-indicators`) are preferable to monolithic branches.
- Smaller, focused PRs are easier to review and less likely to introduce regressions.

### 6. Automated Visual Regression Testing

Consider implementing visual regression testing (e.g., BackstopJS, Percy, or Chromatic) that captures screenshots of key pages/components before and after changes. This would automatically flag visual regressions like the menu hover state issue.

---

## Appendix A: File Paths Reference

| File | Purpose | Status |
|------|---------|--------|
| `web/themes/custom/concorde_bs/concorde_bs.info.yml` | Theme definition, global library loading | Library disabled (commented out) |
| `web/themes/custom/concorde_bs/concorde_bs.libraries.yml` | Library definitions | `custom` library still defined (lines 169-177) |
| `web/themes/custom/concorde_bs/css/a11y.css` | Problematic CSS file | Exists but not loaded |
| `web/themes/custom/concorde_bs/js/a11y.js` | Problematic JS file | Exists but not loaded |
| `web/themes/custom/concorde_bs/src/js/components/ada.js` | Properly integrated ADA JS | Active (bundled into scripts.min.js) |
| `web/themes/custom/concorde_bs/src/js/scripts.js` | JS entry point | Imports ada.js correctly |
| `web/themes/custom/concorde_bs/src/scss/_components.scss` | SCSS component imports | Where new a11y partial should be added |
| `web/themes/custom/concorde_bs/src/scss/components/_navbar.scss` | Navbar/megamenu styles | Reference for proper SCSS patterns |
| `web/themes/custom/concorde_bs/src/js/components/dropdown-megamenu.js` | Megamenu JS | Uses mouseenter events affected by overflow changes |
| `web/themes/custom/concorde_bs/src/js/components/offcanvas-submenu.js` | Offcanvas submenu JS | Manages nested offcanvas panels |
| `web/themes/custom/concorde_bs/build/config.js` | Build path configuration | Defines src/output directories |
| `web/themes/custom/concorde_bs/package.json` | Node.js build scripts | `npm run build` for compilation |

---

## Appendix B: Git Commit Details

### Problematic Commits (ADA Sprint)

```
0eb174a17  Feb 23  Prince  Fix breadcrumb issues and add js/css file
           INTRODUCED: css/a11y.css, js/a11y.js, custom library, enabled globally

0afe9be17  Feb 23  Prince  submenu headings
           MODIFIED: a11y.css (chat widget height tweak)

1bee7b629  Feb 27  Prince  Change focus indicator color and fix 9 issues
           MODIFIED: a11y.css (+18 lines: overflow, focus styles)
           MODIFIED: a11y.js (+10 lines: jQuery, chat label, tabindex removal)
           ** Introduced menu hover/click bug **

f2fb99a39  Feb 27  Prince  visible Schedule tour link
           MODIFIED: a11y.css (+8 lines: chat input border, sticky footer d-none override)
           ** Introduced sticky footer CTA bug **

47d3aba1a  Feb 27  Prince  overflow on sticky footer
           MODIFIED: a11y.css (+4 lines: sticky footer overflow:auto)
           ** Introduced horizontal scrollbar bug **

69f414f9d  Feb 27  Prince  adjust with sticky footer
           MODIFIED: a11y.css (+5 lines: sticky footer width override)
           ** Worsened horizontal scrollbar bug **
```

### Resolution Commit

```
a811fb231  Mar 02  Dan Vilione  Update composer dependencies, fix block configuration...
           MODIFIED: concorde_bs.info.yml (commented out concorde_bs/custom)
           ** Disabled the problematic library **
```

---

*End of Report*

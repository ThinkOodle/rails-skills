# CSS Architecture Reference

Detailed patterns, examples, and edge cases for the css-architecture skill.

## Complete File Organization

### Directory Structure

```
app/assets/stylesheets/
├── _global.css              # Layer declarations + design tokens
├── application.css          # Entry point (imports or Rails manifest)
├── reset.css                # CSS reset
├── base.css                 # Element defaults (no classes)
├── utilities.css            # Single-purpose utility classes
├── components/
│   ├── app-layout.css       # Main layout shell (header, sidebar, main)
│   ├── forms.css            # Inputs, labels, buttons, form layout
│   ├── cards.css            # Card component + variants
│   ├── badges.css           # Badge variants
│   ├── alerts.css           # Alert/flash messages
│   ├── tables.css           # Table styles
│   ├── modals.css           # Modal/dialog patterns
│   ├── nav.css              # Navigation components
│   └── [feature].css        # Feature-specific components
└── pages/                   # Optional: page-specific layout (thin!)
    ├── dashboard.css
    └── settings.css
```

### What Goes Where

| File | Contains | Does NOT Contain |
|------|----------|------------------|
| `_global.css` | `@layer` declaration, `:root` tokens, raw palette | Component styles, classes |
| `reset.css` | Browser normalization | Design decisions |
| `base.css` | Element defaults (`body`, `h1-h6`, `a`, `input`) | Classes, component styles |
| `utilities.css` | Single-purpose `.mt-4`, `.hidden`, `.flex` | Multi-property classes |
| `components/*.css` | Reusable UI components | Page-specific overrides |
| `pages/*.css` | Page-specific layout grids | Component style overrides |

### application.css Entry Point

With Rails asset pipeline (Propshaft or Sprockets), the entry point typically imports everything. With `cssbundling-rails` or import maps, it varies. The key rule: `_global.css` loads first.

```css
/* application.css — if using CSS imports */
@import "_global.css";
@import "reset.css";
@import "base.css";
@import "components/app-layout.css";
@import "components/forms.css";
@import "components/cards.css";
@import "components/badges.css";
@import "components/alerts.css";
@import "components/tables.css";
@import "utilities.css";
```

---

## Complete Design Token System

### Raw Color Palette (OKLCH)

OKLCH provides perceptually uniform colors — equal lightness steps look equal to the human eye.

```css
:root {
  /* Zinc — neutral scale */
  --color-zinc-50:  oklch(0.985 0 0);
  --color-zinc-100: oklch(0.967 0.001 286.375);
  --color-zinc-200: oklch(0.92 0.004 286.32);
  --color-zinc-300: oklch(0.87 0.006 286.286);
  --color-zinc-400: oklch(0.707 0.022 261);
  --color-zinc-500: oklch(0.552 0.016 285.938);
  --color-zinc-600: oklch(0.442 0.017 285.786);
  --color-zinc-700: oklch(0.37 0.013 285.805);
  --color-zinc-800: oklch(0.274 0.006 286.033);
  --color-zinc-900: oklch(0.21 0.006 285.885);
  --color-zinc-950: oklch(0.141 0.005 285.823);

  /* Blue — primary/brand */
  --color-blue-50:  oklch(0.97 0.014 254);
  --color-blue-100: oklch(0.932 0.032 255);
  --color-blue-200: oklch(0.882 0.059 254);
  --color-blue-400: oklch(0.707 0.165 254.624);
  --color-blue-500: oklch(0.623 0.214 259.815);
  --color-blue-600: oklch(0.546 0.245 262.881);
  --color-blue-700: oklch(0.488 0.243 264.376);

  /* Green — positive/success */
  --color-green-50:  oklch(0.962 0.044 156);
  --color-green-500: oklch(0.586 0.154 149);
  --color-green-600: oklch(0.517 0.148 149);
  --color-green-800: oklch(0.345 0.108 148);
  --color-green-950: oklch(0.208 0.075 152);

  /* Red — negative/danger */
  --color-red-50:  oklch(0.971 0.013 17);
  --color-red-500: oklch(0.577 0.245 27.325);
  --color-red-600: oklch(0.528 0.229 25);
  --color-red-800: oklch(0.395 0.165 25);
  --color-red-950: oklch(0.258 0.092 26);

  /* Yellow — warning */
  --color-yellow-50:  oklch(0.987 0.026 102);
  --color-yellow-500: oklch(0.795 0.184 86);
  --color-yellow-600: oklch(0.681 0.162 75);
  --color-yellow-950: oklch(0.271 0.067 70);

  /* Purple — feature/accent (optional) */
  --color-purple-50:  oklch(0.977 0.014 308);
  --color-purple-500: oklch(0.553 0.235 303);
  --color-purple-600: oklch(0.496 0.265 301);
  --color-purple-950: oklch(0.224 0.114 303);
}
```

### Semantic Color Tokens

Semantic tokens map intent to palette values. Components only ever reference these.

```css
:root {
  color-scheme: light dark;

  /* === Surfaces (backgrounds) === */
  --color-canvas:          light-dark(var(--color-zinc-50), var(--color-zinc-900));
  --color-surface:         light-dark(white, var(--color-zinc-800));
  --color-surface-raised:  light-dark(white, var(--color-zinc-700));
  --color-surface-muted:   light-dark(var(--color-zinc-100), var(--color-zinc-800));
  --color-surface-overlay: light-dark(white, var(--color-zinc-800));

  /* === Ink (text) === */
  --color-ink:        light-dark(var(--color-zinc-900), var(--color-zinc-50));
  --color-ink-muted:  light-dark(var(--color-zinc-500), var(--color-zinc-400));
  --color-ink-subtle: light-dark(var(--color-zinc-400), var(--color-zinc-500));

  /* === Borders === */
  --color-border:        light-dark(var(--color-zinc-200), var(--color-zinc-700));
  --color-border-muted:  light-dark(var(--color-zinc-100), var(--color-zinc-800));
  --color-border-strong: light-dark(var(--color-zinc-400), var(--color-zinc-600));

  /* === Interactive === */
  --color-primary:       light-dark(var(--color-blue-600), var(--color-blue-500));
  --color-primary-hover: light-dark(var(--color-blue-700), var(--color-blue-400));
  --color-link:          light-dark(var(--color-blue-600), var(--color-blue-400));
  --color-focus-ring:    light-dark(var(--color-blue-200), var(--color-blue-700));

  /* === State: Positive === */
  --color-positive:        light-dark(var(--color-green-600), var(--color-green-500));
  --color-positive-canvas: light-dark(var(--color-green-50), var(--color-green-950));
  --color-positive-border: light-dark(var(--color-green-200), var(--color-green-800));

  /* === State: Negative === */
  --color-negative:        light-dark(var(--color-red-600), var(--color-red-500));
  --color-negative-canvas: light-dark(var(--color-red-50), var(--color-red-950));
  --color-negative-border: light-dark(var(--color-red-200), var(--color-red-800));

  /* === State: Warning === */
  --color-warning:        light-dark(var(--color-yellow-600), var(--color-yellow-500));
  --color-warning-canvas: light-dark(var(--color-yellow-50), var(--color-yellow-950));
  --color-warning-border: light-dark(var(--color-yellow-200), var(--color-yellow-800));
}
```

### Spacing Scale

```css
:root {
  --space-0:  0;
  --space-px: 1px;
  --space-0.5: 0.125rem;  /* 2px */
  --space-1:  0.25rem;    /* 4px */
  --space-1.5: 0.375rem;  /* 6px */
  --space-2:  0.5rem;     /* 8px */
  --space-3:  0.75rem;    /* 12px */
  --space-4:  1rem;       /* 16px */
  --space-5:  1.25rem;    /* 20px */
  --space-6:  1.5rem;     /* 24px */
  --space-8:  2rem;       /* 32px */
  --space-10: 2.5rem;     /* 40px */
  --space-12: 3rem;       /* 48px */
  --space-16: 4rem;       /* 64px */
  --space-20: 5rem;       /* 80px */
}
```

### Typography Scale

```css
:root {
  /* Font families */
  --font-sans: system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  --font-mono: ui-monospace, "SF Mono", "Cascadia Code", monospace;

  /* Sizes */
  --text-xs:   0.75rem;   /* 12px */
  --text-sm:   0.875rem;  /* 14px */
  --text-base: 1rem;      /* 16px */
  --text-lg:   1.125rem;  /* 18px */
  --text-xl:   1.25rem;   /* 20px */
  --text-2xl:  1.5rem;    /* 24px */
  --text-3xl:  1.875rem;  /* 30px */
  --text-4xl:  2.25rem;   /* 36px */

  /* Weights */
  --font-normal:   400;
  --font-medium:   500;
  --font-semibold: 600;
  --font-bold:     700;

  /* Line heights */
  --leading-tight:  1.25;
  --leading-normal: 1.5;
  --leading-relaxed: 1.625;
}
```

### Border Radius and Transitions

```css
:root {
  /* Radii */
  --radius-sm:   0.25rem;  /* 4px */
  --radius-md:   0.375rem; /* 6px */
  --radius-lg:   0.5rem;   /* 8px */
  --radius-xl:   0.75rem;  /* 12px */
  --radius-2xl:  1rem;     /* 16px */
  --radius-full: 9999px;

  /* Transitions */
  --duration-fast:   150ms;
  --duration-normal: 250ms;
  --duration-slow:   350ms;
  --ease-out:  cubic-bezier(0.16, 1, 0.3, 1);
  --ease-in:   cubic-bezier(0.7, 0, 0.84, 0);

  /* Shadows (use sparingly — prefer borders in dark mode) */
  --shadow-sm: 0 1px 2px rgb(0 0 0 / 0.04);
  --shadow-md: 0 2px 4px rgb(0 0 0 / 0.06);
}
```

---

## Component Patterns

### Button Component

```css
/* components/forms.css or components/buttons.css */
@layer components {
  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: var(--space-2);
    padding: var(--space-2) var(--space-4);
    font-size: var(--text-sm);
    font-weight: var(--font-medium);
    line-height: var(--leading-tight);
    border-radius: var(--radius-lg);
    border: 1px solid transparent;
    cursor: pointer;
    transition: all var(--duration-fast) var(--ease-out);
  }

  .btn:focus-visible {
    outline: 2px solid var(--color-focus-ring);
    outline-offset: 2px;
  }

  /* Variants */
  .btn-primary {
    background: var(--color-primary);
    color: white;
  }
  .btn-primary:hover {
    background: var(--color-primary-hover);
  }

  .btn-secondary {
    background: var(--color-surface);
    border-color: var(--color-border);
    color: var(--color-ink);
  }
  .btn-secondary:hover {
    background: var(--color-surface-muted);
    border-color: var(--color-border-strong);
  }

  .btn-danger {
    background: var(--color-negative);
    color: white;
  }

  .btn-ghost {
    background: transparent;
    color: var(--color-ink-muted);
  }
  .btn-ghost:hover {
    background: var(--color-surface-muted);
    color: var(--color-ink);
  }

  /* Sizes */
  .btn-sm {
    padding: var(--space-1) var(--space-2);
    font-size: var(--text-xs);
  }
  .btn-lg {
    padding: var(--space-3) var(--space-6);
    font-size: var(--text-base);
  }

  /* States */
  .btn:disabled,
  .btn.is-loading {
    opacity: 0.6;
    pointer-events: none;
  }
}
```

### Card Component

```css
/* components/cards.css */
@layer components {
  .card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-lg);
    padding: var(--space-6);
  }

  .card-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding-bottom: var(--space-4);
    border-bottom: 1px solid var(--color-border-muted);
    margin-bottom: var(--space-4);
  }

  .card-title {
    font-size: var(--text-lg);
    font-weight: var(--font-semibold);
    color: var(--color-ink);
  }

  .card-body {
    color: var(--color-ink);
  }

  .card-footer {
    display: flex;
    align-items: center;
    gap: var(--space-3);
    padding-top: var(--space-4);
    border-top: 1px solid var(--color-border-muted);
    margin-top: var(--space-4);
  }

  /* Variants */
  .card-sm { padding: var(--space-4); }
  .card-lg { padding: var(--space-8); }
  .card-flush { padding: 0; }
  .card-flush .card-header,
  .card-flush .card-body,
  .card-flush .card-footer { padding-left: var(--space-6); padding-right: var(--space-6); }

  /* Interactive card */
  .card-interactive {
    transition: border-color var(--duration-fast) var(--ease-out);
    cursor: pointer;
  }
  .card-interactive:hover {
    border-color: var(--color-border-strong);
  }
}
```

### Badge Component

```css
/* components/badges.css */
@layer components {
  .badge {
    display: inline-flex;
    align-items: center;
    gap: var(--space-1);
    padding: var(--space-0.5) var(--space-2);
    font-size: var(--text-xs);
    font-weight: var(--font-medium);
    border-radius: var(--radius-full);
    line-height: var(--leading-tight);
  }

  .badge-neutral {
    color: var(--color-ink-muted);
    background: var(--color-surface-muted);
  }

  .badge-positive {
    color: var(--color-positive);
    background: var(--color-positive-canvas);
  }

  .badge-negative {
    color: var(--color-negative);
    background: var(--color-negative-canvas);
  }

  .badge-warning {
    color: var(--color-warning);
    background: var(--color-warning-canvas);
  }

  /* Using color-mix for custom badge colors */
  .badge-feature {
    color: var(--color-purple-600);
    background: color-mix(in oklch, var(--color-purple-500) 15%, var(--color-surface));
  }
}
```

### Alert Component

```css
/* components/alerts.css */
@layer components {
  .alert {
    padding: var(--space-4);
    border-radius: var(--radius-lg);
    border: 1px solid;
    font-size: var(--text-sm);
  }

  .alert-info {
    color: var(--color-primary);
    background: light-dark(var(--color-blue-50), var(--color-blue-700));
    border-color: light-dark(var(--color-blue-200), var(--color-blue-700));
  }

  .alert-success {
    color: var(--color-positive);
    background: var(--color-positive-canvas);
    border-color: var(--color-positive-border);
  }

  .alert-danger {
    color: var(--color-negative);
    background: var(--color-negative-canvas);
    border-color: var(--color-negative-border);
  }

  .alert-warning {
    color: var(--color-warning);
    background: var(--color-warning-canvas);
    border-color: var(--color-warning-border);
  }
}
```

### Form Component

```css
/* components/forms.css */
@layer components {
  .form-group {
    display: flex;
    flex-direction: column;
    gap: var(--space-1.5);
  }

  .form-label {
    font-size: var(--text-sm);
    font-weight: var(--font-medium);
    color: var(--color-ink);
  }

  .form-input {
    padding: var(--space-2) var(--space-3);
    font-size: var(--text-sm);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    background: var(--color-surface);
    color: var(--color-ink);
    transition: border-color var(--duration-fast) var(--ease-out);
  }

  .form-input:focus {
    outline: none;
    border-color: var(--color-primary);
    box-shadow: 0 0 0 3px var(--color-focus-ring);
  }

  .form-input::placeholder {
    color: var(--color-ink-subtle);
  }

  .form-help {
    font-size: var(--text-xs);
    color: var(--color-ink-muted);
  }

  .form-error {
    font-size: var(--text-xs);
    color: var(--color-negative);
  }

  .form-input.is-invalid {
    border-color: var(--color-negative);
  }

  /* Checkbox / radio */
  .form-check {
    display: flex;
    align-items: center;
    gap: var(--space-2);
  }

  /* Form layout */
  .form-stack {
    display: flex;
    flex-direction: column;
    gap: var(--space-4);
  }

  .form-row {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: var(--space-4);
  }

  .form-actions {
    display: flex;
    align-items: center;
    gap: var(--space-3);
    padding-top: var(--space-4);
  }
}
```

### Table Component

```css
/* components/tables.css */
@layer components {
  .table-container {
    overflow-x: auto;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-lg);
  }

  .table {
    width: 100%;
    border-collapse: collapse;
    font-size: var(--text-sm);
  }

  .table th {
    text-align: left;
    padding: var(--space-3) var(--space-4);
    font-weight: var(--font-medium);
    color: var(--color-ink-muted);
    background: var(--color-surface-muted);
    border-bottom: 1px solid var(--color-border);
  }

  .table td {
    padding: var(--space-3) var(--space-4);
    border-bottom: 1px solid var(--color-border-muted);
    color: var(--color-ink);
  }

  .table tbody tr:last-child td {
    border-bottom: none;
  }

  .table-striped tbody tr:nth-child(even) {
    background: var(--color-surface-muted);
  }

  .table-hover tbody tr:hover {
    background: var(--color-surface-muted);
  }
}
```

---

## Dark Mode Deep Dive

### How light-dark() Works

`light-dark()` is a CSS function that returns its first argument in light mode and its second in dark mode. It requires `color-scheme` to be set on an ancestor.

```css
:root {
  color-scheme: light dark;  /* Required — enables light-dark() */
}

/* light-dark(light-value, dark-value) */
--color-surface: light-dark(white, var(--color-zinc-800));
```

The browser determines "light" or "dark" based on:
1. The `color-scheme` property on the element or ancestor
2. The user's OS preference (`prefers-color-scheme`)
3. Manual override via `data-theme` attribute

### Manual Theme Toggle Pattern

```html
<!-- In <head>, before any CSS loads -->
<script>
  // Apply saved theme before first paint to prevent flash
  const theme = localStorage.getItem('theme');
  if (theme) document.documentElement.dataset.theme = theme;
</script>
```

```css
/* Override color-scheme with data attribute */
html[data-theme="light"] { color-scheme: light; }
html[data-theme="dark"]  { color-scheme: dark; }
```

```javascript
// Toggle function
function toggleTheme() {
  const html = document.documentElement;
  const current = html.dataset.theme;
  const next = current === 'dark' ? 'light' : 'dark';
  html.dataset.theme = next;
  localStorage.setItem('theme', next);
}

// Reset to system preference
function resetTheme() {
  delete document.documentElement.dataset.theme;
  localStorage.removeItem('theme');
}
```

**Stimulus controller version:**

```javascript
// app/javascript/controllers/theme_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  toggle() {
    const html = document.documentElement
    const current = html.dataset.theme
    const next = current === "dark" ? "light" : "dark"
    html.dataset.theme = next
    localStorage.setItem("theme", next)
  }

  reset() {
    delete document.documentElement.dataset.theme
    localStorage.removeItem("theme")
  }
}
```

### Shadows in Dark Mode

Shadows are nearly invisible against dark backgrounds. Use elevation through borders and surface tones instead.

```css
/* Light mode mental model: elevation = shadows */
/* Dark mode mental model: elevation = lighter surfaces */

.card {
  /* Don't rely on shadows for structure */
  border: 1px solid var(--color-border);
}

/* If you must use shadows, keep them very subtle */
.card-elevated {
  box-shadow: var(--shadow-sm);  /* 0 1px 2px rgb(0 0 0 / 0.04) */
}

/* Elevation via surface color is better */
.surface-base   { background: var(--color-canvas); }      /* Lowest */
.surface-card   { background: var(--color-surface); }      /* Middle */
.surface-raised { background: var(--color-surface-raised); } /* Highest */
```

### color-mix() for Tinted Backgrounds

`color-mix()` creates tinted backgrounds that work in both modes:

```css
.badge-feature {
  color: var(--color-purple-600);
  /* Mix 15% of the accent color with the surface color */
  background: color-mix(in oklch, var(--color-purple-500) 15%, var(--color-surface));
}

/* Hover state — slightly more saturated */
.tag:hover {
  background: color-mix(in oklch, var(--color-primary) 20%, var(--color-surface));
}
```

### State Colors Need Three Tokens

Every state (success, error, warning) needs foreground, background, and border:

```css
:root {
  /* Each state gets a complete set */
  --color-positive:        light-dark(var(--color-green-600), var(--color-green-500));
  --color-positive-canvas: light-dark(var(--color-green-50), var(--color-green-950));
  --color-positive-border: light-dark(var(--color-green-200), var(--color-green-800));
}

.alert-success {
  color: var(--color-positive);
  background: var(--color-positive-canvas);
  border: 1px solid var(--color-positive-border);
}
```

### Browser Fallback

For browsers that don't support `light-dark()` (pre-2024):

```css
/* Fallback: define defaults, then override in media query */
:root {
  --color-surface: white;
  --color-ink: #18181b;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-surface: #27272a;
    --color-ink: #fafafa;
  }
}

/* Modern browsers: use light-dark() — overrides the fallback */
@supports (color: light-dark(red, red)) {
  :root {
    --color-surface: light-dark(white, #27272a);
    --color-ink: light-dark(#18181b, #fafafa);
  }
}
```

---

## CSS Layers Deep Dive

### Layer Declaration and Priority

```css
/* _global.css — must be the very first rule */
@layer reset, base, components, utilities;
```

This declares cascade order. Rules in later layers beat earlier layers regardless of specificity.

```css
/* utilities layer always beats components, even if component has higher specificity */
@layer components {
  .card .title { color: red; }     /* Specificity: 0-2-0 */
}

@layer utilities {
  .text-blue { color: blue; }      /* Specificity: 0-1-0 — but WINS because layer is higher */
}
```

### Layer Assignment Patterns

**Inline (preferred for single files):**
```css
@layer components {
  .card { ... }
  .card-header { ... }
}
```

**Import-based (for large projects):**
```css
@import "reset.css" layer(reset);
@import "base.css" layer(base);
@import "components/cards.css" layer(components);
@import "utilities.css" layer(utilities);
```

### Unlayered CSS

CSS not in any layer has highest priority. Avoid this — it defeats the purpose of layers.

```css
/* This beats EVERYTHING, including utilities. Don't do this. */
.card { background: red; }

/* This respects the cascade properly */
@layer components {
  .card { background: var(--color-surface); }
}
```

### Third-Party CSS in Layers

Wrap third-party CSS in a layer to control its priority:

```css
@layer reset, vendor, base, components, utilities;

@import "normalize.css" layer(reset);
@import "some-library.css" layer(vendor);
```

---

## Responsive Patterns

### Mobile-First Breakpoints

```css
/* Base: mobile */
.dashboard-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--space-4);
}

/* Tablet */
@media (min-width: 768px) {
  .dashboard-grid {
    grid-template-columns: repeat(2, 1fr);
    gap: var(--space-6);
  }
}

/* Desktop */
@media (min-width: 1024px) {
  .dashboard-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### Container Queries (Modern Alternative)

```css
.card-grid-container {
  container-type: inline-size;
}

.card-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: var(--space-4);
}

@container (min-width: 600px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

@container (min-width: 900px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### Fluid Typography

```css
:root {
  /* Fluid: 1rem at 320px, 1.25rem at 1280px */
  --text-fluid-base: clamp(1rem, 0.9rem + 0.26vw, 1.25rem);
  --text-fluid-lg: clamp(1.125rem, 1rem + 0.52vw, 1.5rem);
  --text-fluid-xl: clamp(1.5rem, 1.2rem + 1.04vw, 2.25rem);
}
```

### Common Layout Patterns

```css
/* Sidebar + main content */
@layer components {
  .app-layout {
    display: grid;
    grid-template-columns: 1fr;
    min-height: 100dvh;
  }

  @media (min-width: 768px) {
    .app-layout {
      grid-template-columns: 16rem 1fr;
    }
  }

  .app-sidebar {
    background: var(--color-surface);
    border-right: 1px solid var(--color-border);
    padding: var(--space-4);
  }

  .app-main {
    padding: var(--space-6);
    max-width: 80rem;
  }
}

/* Stack with consistent spacing */
.stack > * + * {
  margin-top: var(--space-4);
}

/* Cluster (flex wrap with gap) */
.cluster {
  display: flex;
  flex-wrap: wrap;
  gap: var(--space-3);
  align-items: center;
}
```

---

## Utility Classes Reference

### Recommended Utility Set

Keep the utility set small. Only add what you actually use.

```css
@layer utilities {
  /* === Spacing === */
  .m-0  { margin: 0; }
  .mt-1 { margin-top: var(--space-1); }
  .mt-2 { margin-top: var(--space-2); }
  .mt-3 { margin-top: var(--space-3); }
  .mt-4 { margin-top: var(--space-4); }
  .mt-6 { margin-top: var(--space-6); }
  .mt-8 { margin-top: var(--space-8); }
  .mb-2 { margin-bottom: var(--space-2); }
  .mb-4 { margin-bottom: var(--space-4); }
  .mb-6 { margin-bottom: var(--space-6); }
  .ml-auto { margin-left: auto; }
  .p-0  { padding: 0; }
  .p-2  { padding: var(--space-2); }
  .p-4  { padding: var(--space-4); }
  .p-6  { padding: var(--space-6); }
  .px-4 { padding-left: var(--space-4); padding-right: var(--space-4); }
  .py-2 { padding-top: var(--space-2); padding-bottom: var(--space-2); }
  .gap-2 { gap: var(--space-2); }
  .gap-3 { gap: var(--space-3); }
  .gap-4 { gap: var(--space-4); }
  .gap-6 { gap: var(--space-6); }

  /* === Typography === */
  .text-xs     { font-size: var(--text-xs); }
  .text-sm     { font-size: var(--text-sm); }
  .text-base   { font-size: var(--text-base); }
  .text-lg     { font-size: var(--text-lg); }
  .text-xl     { font-size: var(--text-xl); }
  .text-2xl    { font-size: var(--text-2xl); }
  .font-medium   { font-weight: var(--font-medium); }
  .font-semibold { font-weight: var(--font-semibold); }
  .font-bold     { font-weight: var(--font-bold); }
  .text-muted  { color: var(--color-ink-muted); }
  .text-subtle { color: var(--color-ink-subtle); }
  .text-center { text-align: center; }
  .text-right  { text-align: right; }
  .truncate    { overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }

  /* === Layout === */
  .flex   { display: flex; }
  .grid   { display: grid; }
  .block  { display: block; }
  .inline { display: inline; }
  .inline-flex { display: inline-flex; }
  .hidden { display: none; }

  .items-center   { align-items: center; }
  .items-start    { align-items: flex-start; }
  .items-end      { align-items: flex-end; }
  .justify-center { justify-content: center; }
  .justify-between { justify-content: space-between; }
  .justify-end    { justify-content: flex-end; }
  .flex-col       { flex-direction: column; }
  .flex-wrap      { flex-wrap: wrap; }
  .flex-1         { flex: 1; }
  .flex-shrink-0  { flex-shrink: 0; }

  /* === Sizing === */
  .w-full { width: 100%; }
  .max-w-sm  { max-width: 24rem; }
  .max-w-md  { max-width: 28rem; }
  .max-w-lg  { max-width: 32rem; }
  .max-w-xl  { max-width: 36rem; }
  .max-w-2xl { max-width: 42rem; }

  /* === Borders === */
  .rounded     { border-radius: var(--radius-md); }
  .rounded-lg  { border-radius: var(--radius-lg); }
  .rounded-full { border-radius: var(--radius-full); }
  .border      { border: 1px solid var(--color-border); }
  .border-b    { border-bottom: 1px solid var(--color-border); }

  /* === Visibility === */
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }
  .opacity-0   { opacity: 0; }
  .opacity-50  { opacity: 0.5; }
  .opacity-100 { opacity: 1; }

  /* === Position === */
  .relative { position: relative; }
  .absolute { position: absolute; }
  .sticky   { position: sticky; }
  .top-0    { top: 0; }
  .right-0  { right: 0; }
  .inset-0  { inset: 0; }

  /* === Misc === */
  .cursor-pointer { cursor: pointer; }
  .overflow-hidden { overflow: hidden; }
  .overflow-auto   { overflow: auto; }
}
```

---

## Rails Integration Patterns

### ERB Component Usage

```erb
<%# Compose components + utilities in views %>
<div class="card">
  <div class="card-header">
    <h3 class="card-title"><%= @project.name %></h3>
    <span class="badge badge-positive"><%= @project.status %></span>
  </div>
  <div class="card-body">
    <p class="text-muted text-sm"><%= @project.description %></p>
  </div>
  <div class="card-footer">
    <%= link_to "Edit", edit_project_path(@project), class: "btn btn-secondary btn-sm" %>
    <%= link_to "View", @project, class: "btn btn-primary btn-sm" %>
  </div>
</div>
```

### Flash Messages

```erb
<%# app/views/layouts/_flash.html.erb %>
<% flash.each do |type, message| %>
  <% alert_class = case type
    when "notice" then "alert-success"
    when "alert"  then "alert-danger"
    else "alert-info"
  end %>
  <div class="alert <%= alert_class %>" role="alert">
    <%= message %>
  </div>
<% end %>
```

### Form Builder Integration

```erb
<%= form_with(model: @user, class: "form-stack") do |f| %>
  <div class="form-group">
    <%= f.label :name, class: "form-label" %>
    <%= f.text_field :name, class: "form-input" %>
  </div>

  <div class="form-group">
    <%= f.label :email, class: "form-label" %>
    <%= f.email_field :email, class: "form-input" %>
    <span class="form-help">We'll never share your email.</span>
  </div>

  <div class="form-actions">
    <%= f.submit "Save", class: "btn btn-primary" %>
    <%= link_to "Cancel", users_path, class: "btn btn-secondary" %>
  </div>
<% end %>
```

### ViewComponent Integration

```ruby
# app/components/card_component.rb
class CardComponent < ViewComponent::Base
  def initialize(variant: nil, interactive: false)
    @variant = variant
    @interactive = interactive
  end

  def classes
    class_names(
      "card",
      "card-#{@variant}" => @variant,
      "card-interactive" => @interactive
    )
  end
end
```

```erb
<%# app/components/card_component.html.erb %>
<div class="<%= classes %>">
  <%= content %>
</div>
```

---

## Naming Convention Reference

| Pattern | Example | Use For |
|---------|---------|---------|
| `.component` | `.card`, `.btn`, `.badge` | Base component |
| `.component-sub` | `.card-header`, `.form-label` | Structural child |
| `.component-variant` | `.card-sm`, `.btn-primary` | Visual variant |
| `.is-state` | `.is-loading`, `.is-active` | Temporary state (global) |
| `.page-element` | `.dashboard-grid` | Page-specific layout |
| `.utility` | `.mt-4`, `.hidden`, `.flex` | Single-purpose override |

### Naming Rules

1. **No generic names** — `.container`, `.title`, `.item` WILL collide
2. **Prefix page-specific** — `.dashboard-grid`, `.settings-sidebar`
3. **Hyphenate everything** — `.card-header` not `.cardHeader` or `.card_header`
4. **Variants extend the base name** — `.btn-primary` not `.primary-btn`
5. **State classes start with `is-` or `has-`** — `.is-loading`, `.has-error`

---

## Anti-Pattern Examples

### ❌ Styling Components From Page CSS

```css
/* BAD — dashboard.css overriding card styles */
.dashboard .card { padding: 2rem; }
.dashboard .card .title { font-size: 24px; }

/* GOOD — card variant in cards.css */
.card-lg { padding: var(--space-8); }
.card-title-lg { font-size: var(--text-2xl); }
```

### ❌ Deep Nesting

```css
/* BAD */
.dashboard .sidebar .nav .item .link { color: blue; }

/* GOOD */
.nav-link { color: var(--color-link); }
```

### ❌ Using !important

```css
/* BAD */
.card { padding: 2rem !important; }

/* GOOD — use @layer */
@layer utilities {
  .p-8 { padding: var(--space-8); }
}
```

### ❌ Inline Styles in ERB

```erb
<%# BAD %>
<div style="margin-top: 16px; padding: 24px; background: white;">

<%# GOOD %>
<div class="card mt-4">
```

### ❌ Framework-Dependent Markup

```erb
<%# BAD — tightly coupled to Tailwind %>
<div class="bg-white dark:bg-zinc-800 border border-zinc-200 dark:border-zinc-700 rounded-lg p-6">

<%# GOOD — semantic classes, framework-free %>
<div class="card">
```

### ❌ Magic Numbers

```css
/* BAD */
.card { padding: 23px; margin-top: 17px; border-radius: 7px; }

/* GOOD */
.card { padding: var(--space-6); margin-top: var(--space-4); border-radius: var(--radius-lg); }
```

### ❌ Raw Colors in Components

```css
/* BAD */
.alert-success { background: #dcfce7; color: #166534; }

/* GOOD */
.alert-success { background: var(--color-positive-canvas); color: var(--color-positive); }
```

---

## Checklist: Adding a New Component

1. Create `app/assets/stylesheets/components/[name].css`
2. Wrap all rules in `@layer components { }`
3. Define base class: `.component-name`
4. Add subcomponents: `.component-name-header`, `.component-name-body`
5. Add variants: `.component-name-sm`, `.component-name-primary`
6. Use only design tokens (no hardcoded values)
7. Use semantic color tokens (not raw palette)
8. Verify dark mode works (toggle OS preference)
9. Test responsive behavior
10. Import in `application.css` if needed

## Checklist: Adding a New Design Token

1. Add to `_global.css` under `:root`
2. For colors: use `light-dark()` with both mode values
3. For palette values: use OKLCH
4. Name semantically (`--color-surface-elevated`, not `--color-gray-lighter`)
5. Follow existing scale patterns (spacing, typography)
6. Document purpose with a comment if non-obvious

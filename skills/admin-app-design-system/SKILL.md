---
name: admin-app-design-system
description: Use when starting or restyling a data-dense internal/admin web app built with React, Tailwind and shadcn/ui, setting up design tokens, theming, dark mode, the app shell (header, sidebar, tenant selector), or choosing which UI primitives to build because shadcn does not ship them. Also use when a status needs a colour and only destructive exists, when the palette drifts because components hardcode hex values, or when adding a page that must match an existing product. Triggers / gatilhos: design system, design tokens, tema, dark mode, sidebar, shell, shadcn, tailwind, paleta, cores semânticas, layout de app administrativo.
---

# Admin App Design System

## Overview

shadcn/ui gives you unstyled-but-accessible primitives and a token contract. It
does **not** give you the vocabulary a data-dense admin app needs: there is no
`success`, no `warning`, no `info`, only `destructive`. Teams discover this on
the first status badge and start hardcoding `text-green-600`, and from then on
the palette drifts forever.

**Core principle: every colour a component uses must resolve to a semantic token.
The moment a component names a hue, the design system has ended.**

## When to Use

- Bootstrapping a new admin/internal app that must look deliberate, not templated
- Extending shadcn with the states shadcn does not define
- Setting up light/dark theming that a component author cannot accidentally break
- Deciding whether a UI primitive should be built or improvised per-page

**Not for:** marketing sites, consumer apps with a bespoke visual identity, or
projects that already have a mature design system.

## Day-Zero Setup

### 1. Token layer: `src/index.css`

Define light values on `:root`, and redefine **only what changes** under `.dark`.
Values are HSL triplets without the `hsl()` wrapper, so Tailwind can compose them
with opacity modifiers.

```css
:root {
  --background: 210 40% 98%;        /* off-white with a cool cast, not #fff  */
  --foreground: 222 47% 11%;

  --card: 0 0% 100%;
  --card-foreground: 222 47% 11%;

  --primary: 221 83% 53%;           /* the one brand hue                     */
  --primary-foreground: 0 0% 100%;

  --secondary: 216 33% 94%;
  --secondary-foreground: 222 47% 11%;

  --muted: 220 14% 96%;
  --muted-foreground: 220 9% 46%;   /* every secondary label in the product  */

  --accent: 214 32% 91%;
  --accent-foreground: 222 47% 11%;

  /* --- the four states. shadcn ships only the first --- */
  --destructive: 347 80% 46%;  --destructive-foreground: 0 0% 100%;
  --success:     158 64% 36%;  --success-foreground:     0 0% 100%;
  --warning:      38 92% 50%;  --warning-foreground:   222 47% 11%;
  --info:        199 89% 48%;  --info-foreground:        0 0% 100%;

  --border: 214 32% 91%;
  --input: 214 32% 91%;
  --input-background: 220 14% 96%;  /* inputs sit *in* the page, not on it   */
  --ring: 221 83% 53%;

  --radius: 0.625rem;               /* 10px, softer than shadcn's 0.5rem    */
}
```

Note `--warning-foreground` is **dark**. Amber at 50% lightness cannot carry white
text at accessible contrast. Every state token needs its own foreground decided
against its own background, not inherited.

### 2. The sidebar is its own palette

```css
:root {
  --sidebar-background: 222 57% 6%;   /* dark even while the page is light */
  --sidebar-foreground: 210 40% 96%;
  --sidebar-accent: 222 47% 14%;
  --sidebar-border: 222 47% 16%;
}
```

A permanently dark sidebar against a light workspace is a deliberate choice worth
copying: it separates navigation from content without a heavy border, keeps the
brand hue readable at small sizes, and means switching to dark mode changes only
the workspace, the chrome stays put, so the app does not appear to restructure
itself.

Give it its own token group. Reusing `--background` for the sidebar couples two
surfaces that must be free to move independently.

### 3. Depth and emphasis

```css
--gradient-primary: linear-gradient(135deg, hsl(221 83% 53%), hsl(226 71% 40%));
--shadow-sm:   0 1px 2px 0 hsl(222 47% 11% / 0.06);
--shadow-md:   0 8px 18px -12px hsl(222 47% 11% / 0.28);
--shadow-lg:   0 20px 42px -24px hsl(222 47% 11% / 0.35);
--shadow-glow: 0 0 28px hsl(221 83% 53% / 0.25);
```

Shadows are tinted with the foreground hue, never neutral black, that is what
keeps elevation from looking grey and dirty. The large negative spread produces a
soft, wide shadow instead of a hard drop. `--shadow-glow` exists for one job:
marking the single active/primary element on a screen.

### 4. Tailwind wiring: `tailwind.config.ts`

```ts
export default {
  darkMode: ["class"],
  theme: {
    extend: {
      fontFamily: {
        sans: ['"Inter"', 'system-ui', '-apple-system', 'BlinkMacSystemFont',
               '"Segoe UI"', 'sans-serif'],
      },
      colors: {
        primary:     { DEFAULT: "hsl(var(--primary))",     foreground: "hsl(var(--primary-foreground))" },
        success:     { DEFAULT: "hsl(var(--success))",     foreground: "hsl(var(--success-foreground))" },
        warning:     { DEFAULT: "hsl(var(--warning))",     foreground: "hsl(var(--warning-foreground))" },
        info:        { DEFAULT: "hsl(var(--info))",        foreground: "hsl(var(--info-foreground))" },
        destructive: { DEFAULT: "hsl(var(--destructive))", foreground: "hsl(var(--destructive-foreground))" },
        // ...background, foreground, card, muted, accent, border, input, ring
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
    },
  },
} satisfies Config;
```

Inter is the right default for this class of app: it was drawn for UI at small
sizes, its tabular figures keep numeric columns aligned, and it is unremarkable , 
which is what you want when the data is the subject.

## The App Shell

Four files, one responsibility each:

| File | Owns |
|---|---|
| `AppLayout.tsx` | The grid: sidebar + scrolling content region |
| `AppSidebar.tsx` | Navigation tree, collapse state, active route |
| `AppHeader.tsx` | Page identity and page-level actions |
| `AppLayoutContext.ts` | Shared open/collapsed state, without prop drilling |

**Make the header's title required.** It is a one-line type that prevents a whole
category of half-built page:

```ts
interface AppHeaderProps {
  title: string;                 // required, a page without a name is a bug
  description?: string;
  actions?: React.ReactNode;     // page-level buttons, right-aligned
}
```

In a real codebase this caught a page shipped with no title as a compile error
rather than as an empty header in production.

The header itself is sticky, of fixed height, and translucent:

```tsx
<header className="sticky top-0 z-30 border-b border-border
                   bg-background/95 shadow-sm backdrop-blur-md">
  <div className="flex h-16 items-center justify-between gap-4 px-4 lg:px-6">
```

`backdrop-blur-md` with a 95% background is what keeps dense table content
legible as it scrolls under the header. A solid header works too; a transparent
one does not.

**Multi-tenant apps need a tenant selector in the shell**, not on each page.
Put it beside the page title so the answer to "which company am I looking at?" is
always in the same place. Getting this wrong is how someone reads the right
report for the wrong entity.

## Primitives to Build (shadcn Does Not Ship These)

Every data-dense admin app converges on the same missing pieces. Build them once,
early, as real components with a stable API, improvising them per page is how
twelve subtly different filter UIs end up in one product.

| Component | Why it is unavoidable |
|---|---|
| `searchable-select` | A native `<Select>` is unusable past ~20 options. Admin apps routinely have hundreds. |
| `multi-select-searchable` | The same, when the filter is "any of these". |
| `table-pagination` | Server-side paging needs page, size, total, and a size picker as one contract. |
| `column-filter-header` | Per-column filter + sort in the header cell, so filters live where the data is. |
| `active-filters-bar` | Applied filters as removable chips. Without it, users forget a filter is on and mistrust the numbers. |

Suggested contracts, kept deliberately small:

```ts
export interface SearchableSelectOption {
  value: string; label: string;
  description?: string;   // secondary line, codes, context
  badge?: string;         // short status marker
  disabled?: boolean;
}

interface TablePaginationProps {
  page: number; pageSize: number; totalCount: number; totalPages: number;
  onPageChange: (page: number) => void;
  onPageSizeChange?: (size: number) => void;
  pageSizeOptions?: number[];
}

interface ActiveFiltersBarProps {
  filters: Record<string, FilterValue>;
  columnLabels: Record<string, string>;   // keys → human labels
  onClearFilter: (key: string) => void;
  onClearAll: () => void;                 // always offer the escape hatch
}
```

`onClearAll` is not optional in practice. A user who cannot clear every filter in
one action will reload the page, losing everything else they had set up.

## Quick Reference

| Need | Use |
|---|---|
| Positive outcome, approved, reconciled | `bg-success text-success-foreground` |
| Needs attention, over budget, pending | `bg-warning text-warning-foreground` (dark text) |
| Neutral context, informational | `bg-info text-info-foreground` |
| Destructive or failed | `bg-destructive text-destructive-foreground` |
| Secondary label, helper text | `text-muted-foreground` |
| Any surface above the page | `bg-card` + `shadow-sm` |
| The one primary action on screen | `bg-primary` (+ `--shadow-glow` if it must be found instantly) |
| Input field | `bg-input-background border-input` |

## Common Mistakes

| Mistake | Consequence |
|---|---|
| `text-green-600` for a success state | Palette drift; dark mode breaks; nothing can be restyled centrally |
| White text on the warning colour | Fails contrast. Amber needs a dark foreground |
| Reusing `--background` for the sidebar | Two surfaces that must move independently become coupled |
| Neutral black shadows | Elevation reads as grey dirt instead of depth |
| Optional header title | Pages ship nameless; the bug is invisible until production |
| Per-page filter UIs | Twelve inconsistent filters, none of them tested |
| Tenant selector on the page | Users read the right numbers for the wrong entity |
| Defining a colour only inside `.dark` | Light mode falls back to a browser default |

## Red Flags: Stop

- A hex value or a Tailwind colour name (`slate-700`, `green-600`) in a component
- A new state colour that has no `--foreground` partner
- A component reaching into `--sidebar-*` from the content area, or vice versa
- Two pages with visually different filter controls
- A page whose header renders empty

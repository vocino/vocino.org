# Design: TailAdmin as reference, Vocino as source of truth

This project can lean on **TailAdmin React Pro** (e.g. a local checkout such as `tailadmin-react-pro-2.0-main` on your machine) for **patterns**, not for **brand**. Use it to study stack choices, layout composition, table/chart wiring, and form flows. **Visual identity, color, and typography should match [vocino.com](https://vocino.com).**

---

## What to take from TailAdmin

| Borrow | Why |
|--------|-----|
| **Stack shape** | React 19, Vite 6, Tailwind CSS v4 (`@import "tailwindcss"`, `@theme` in CSS), `tailwind-merge`, React Router 7 |
| **Layout ideas** | Sidebar + main content, header bar, responsive collapse—see `src/layout` and page shells |
| **Component recipes** | Tables, filters, modals, date pickers (Flatpickr), charts (ApexCharts)—copy **behavior**, restyle |
| **Data-heavy UX** | How queues, calendars, and dashboards group information (useful for backlog + schedule views later) |

Paths worth skimming (in the TailAdmin repo):

- `src/index.css` — Tailwind v4 theme tokens (replace with Vocino tokens, not the default `brand-*` blues)
- `src/layout/` — app chrome
- `src/pages/` — composed screens (charts, tables)

Do **not** treat TailAdmin’s **Outfit** font or **indigo `brand-500`** palette as canonical for Organizing Vocino.

---

## Vocino style source of truth

**Authoritative reference:** the live site **vocino.com** (Astro-built CSS; hashed bundle path may change on each deploy). The tokens below are taken from that site’s compiled `:root` rules so this repo has a stable checklist even when the asset filename rotates.

### Color tokens (CSS variables)

Use these as the basis for Tailwind `@theme` or CSS variables in the Organizing Vocino frontend.

| Token | Hex / value | Role |
|-------|----------------|------|
| `--bg` | `#0F1419` | Page background |
| `--surface-1` | `#161B22` | Cards, panels |
| `--surface-2` | `#1E242C` | Elevated / nested surfaces |
| `--selection` | `#263040` | Hover / selected states |
| `--border` | `#2B3440` | Borders, dividers |
| `--text-primary` | `#E6EDF3` | Primary text |
| `--text-secondary` | `#9BA7B4` | Secondary text |
| `--text-muted` | `#6B7785` | Muted / helper text |
| `--text-disabled` | `#4B5563` | Disabled |
| `--brand` | `#00CCFF` | Primary accent, links, focus |
| `--accent-blue` | `#4DA3FF` | Link hover, secondary accent |
| `--accent-purple` | `#9D7CFF` | Tertiary accent |
| `--success` | `#3DFA9A` | Success |
| `--warning` | `#E6E87A` | Warning |
| `--attention` | `#FFB86B` | Attention |
| `--danger` | `#FF5C5C` | Error / destructive |
| `--gradient-primary` | `linear-gradient(90deg, #00CCFF 0%, #4DA3FF 100%)` | Hero / highlights |

Supporting details from vocino.com:

- Subtle **dot grid** and **radial** background treatments on the marketing site; the dashboard can use a **flat `--bg`** or a toned-down variant so data UI stays readable.
- **Focus:** visible outline using brand (e.g. `outline: 2px solid var(--brand)`, `outline-offset: 2px`); inputs often use a soft brand ring (`rgba(0, 204, 255, 0.25)`).

### Typography

| Role | Family | Notes |
|------|--------|--------|
| **UI / body** | `Source Sans 3` | Weights 300–700; primary interface and headings on vocino.com |
| **Display / emphasis** | `Source Serif 4` | Optional for marketing-style headlines; use sparingly in dense dashboards |
| **Monospace** | `Source Code Pro` | Labels, metadata, queue IDs, timestamps |

Google Fonts link used on vocino.com (subset as needed):

`https://fonts.googleapis.com/css2?family=Source+Sans+3:wght@300;400;500;600;700&family=Source+Serif+4:wght@400;500;600;700&family=Source+Code+Pro:wght@400;500;600;700&display=swap`

Base body on vocino.com: **~18px** (`1.125rem`), **line-height ~1.6**, **font-smoothing** on.

### Shape, spacing, motion (dashboard-friendly defaults)

From vocino.com component patterns (style guide CSS in the same bundle):

- **Radius:** `12px` buttons and inputs; `16px` cards and larger panels; `8px` for small elements.
- **Transitions:** ~`0.2s ease` for hovers; ~`0.3s–0.4s cubic-bezier(0.4, 0, 0.2, 1)` for larger UI motion—keep dashboard interactions snappy.
- **Cards:** `surface-1` fill, `1px solid border`, lift on hover optional (`translateY(-2px)` + soft shadow) for primary panels only—avoid noise on every row.

---

## Mapping TailAdmin → Vocino in practice

1. **Replace `@theme` colors** in your app’s `index.css`: map TailAdmin’s `--color-brand-*` / gray ramps to Vocino’s `--brand`, `--surface-*`, `--text-*`, etc. Either redefine Tailwind color names to Vocino hex values or use arbitrary values tied to CSS variables (`bg-[var(--surface-1)]`).
2. **Swap the font import** from Outfit to Source Sans 3 / Source Serif 4 / Source Code Pro; set `--font-sans` (and `font-family` on `body`) accordingly.
3. **Keep TailAdmin class names only where convenient**; prefer semantic layout components in your codebase that **embed** Vocino tokens so a future TailAdmin upgrade does not revert the brand.
4. **Charts** (ApexCharts, etc.): set axis, grid, and tooltip colors to `--text-muted`, `--border`, `--surface-2`, and series colors to `--brand` / `--accent-blue` / `--accent-purple` / `--success` / `--danger` as needed.
5. **Dark-first:** vocino.com is a dark UI; Organizing Vocino should default to the same. A light mode can wait until there is a defined Vocino light palette.

---

## Checklist when scaffolding the Organizing Vocino UI

- [ ] Vocino CSS variables (or Tailwind theme) committed in **this** repo—not copied from TailAdmin’s default `@theme` palette.
- [ ] Fonts loaded to match vocino.com.
- [ ] Primary button: `--brand` background, `--bg` (or near-black) text for contrast; focus ring uses brand alpha.
- [ ] Destructive actions use `--danger`, not TailAdmin’s default error red unless it matches `#FF5C5C`.
- [ ] No TailAdmin logo, demo copy, or demo brand colors in production shell.

---

## Summary

| Layer | Source |
|--------|--------|
| Brand, color, type, dark aesthetic | **vocino.com** |
| Dashboard layout, charts, tables, forms, Tailwind v4 structure | **TailAdmin** (reference implementation) |

If vocino.com’s token set changes, update this document and the theme in Organizing Vocino together.

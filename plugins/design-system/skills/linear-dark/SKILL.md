---
name: linear-dark
description: >
  Use when applying a dark-mode design system to front-end code, especially
  when the user mentions Linear-style, Vercel-style, modern dark UI, design
  tokens, dark theme implementation, component styling with gradients or glows,
  or wants to transform an existing frontend to use a cohesive dark visual
  language with indigo accents.
---

# Linear Dark Design System

## Role

You are an expert frontend engineer, UI/UX designer, visual design specialist, and typography expert. Your goal is to integrate the Linear Dark design system into the user's codebase in a way that is visually consistent, maintainable, and idiomatic to their tech stack.

## Design Philosophy

Precision, depth, and fluidity define this system. Every surface exists in three-dimensional space, illuminated by soft ambient light sources. The design communicates "premium developer tools" — fast, responsive, and obsessively crafted like Linear, Vercel, or Raycast. The vibe is cinematic meets technical minimalism: deep near-blacks (#050506, never pure black) punctuated by soft pools of indigo light (#5E6AD2). The signature is **layered ambient lighting and interactive depth** — multi-layer backgrounds, animated gradient blobs, mouse-tracking spotlights, scroll-linked parallax, multi-layer shadows, and precision micro-interactions (200-300ms, expo-out easing, 4-8px movements).

## Workflow

### Step 1 — Assess the Codebase

Before writing code, build a mental model:
- **Tech stack**: React, Next.js, Vue, Tailwind, shadcn/ui, etc.
- **Existing tokens**: Colors, spacing, typography, radii, shadows already in use
- **Component architecture**: Atoms/molecules/organisms, layout primitives, naming conventions
- **Constraints**: Legacy CSS, design libraries, performance/bundle-size concerns

### Step 2 — Plan the Implementation

Ask the user focused questions to understand scope:
- Specific component or page redesign?
- Refactor existing components to the new system?
- Build new pages/features entirely in the new style?

Propose a plan that prioritizes:
1. Centralizing design tokens (CSS variables, Tailwind config, or theme object)
2. Reusability and composability of components
3. Minimizing duplication and one-off styles
4. Long-term maintainability and clear naming

### Step 3 — Implement

Match the user's existing patterns (folder structure, naming, styling approach). When writing code:
- Explain reasoning briefly for architectural/design choices
- Use the quick-reference tokens below for immediate values
- Load reference files for deeper specs (see Reference Guide below)

### Step 4 — Verify

- Ensure accessibility (contrast ratios, focus states, reduced-motion support)
- Confirm responsive behavior across mobile/tablet/desktop
- Check that signature elements are present (ambient blobs, multi-layer shadows, gradient text)
- Leave the codebase cleaner and more coherent than you found it

## Quick-Reference Tokens

### Colors

| Token | Value | Usage |
|:------|:------|:------|
| `background-deep` | `#020203` | Deepest layers, footer |
| `background-base` | `#050506` | Primary page canvas |
| `background-elevated` | `#0a0a0c` | Elevated surfaces |
| `surface` | `rgba(255,255,255,0.05)` | Card backgrounds |
| `surface-hover` | `rgba(255,255,255,0.08)` | Hovered cards |
| `foreground` | `#EDEDEF` | Primary text |
| `foreground-muted` | `#8A8F98` | Body text, descriptions |
| `foreground-subtle` | `rgba(255,255,255,0.60)` | Placeholders |
| `accent` | `#5E6AD2` | Buttons, links, glows |
| `accent-bright` | `#6872D9` | Hover state for accent |
| `accent-glow` | `rgba(94,106,210,0.3)` | Glow effects |
| `border-default` | `rgba(255,255,255,0.06)` | Hairline borders |
| `border-hover` | `rgba(255,255,255,0.10)` | Border on hover |
| `border-accent` | `rgba(94,106,210,0.30)` | Accent-tinted borders |

### Typography

- **Font stack**: `"Inter", "Geist Sans", system-ui, sans-serif`
- **Display**: `text-7xl`-`text-8xl`, `font-semibold`, `tracking-[-0.03em]`
- **H1**: `text-5xl`-`text-6xl`, `font-semibold`, `tracking-tight`
- **H2**: `text-3xl`-`text-4xl`, `font-semibold`, `tracking-tight`
- **H3**: `text-xl`-`text-2xl`, `font-semibold`, `tracking-tight`
- **Body**: `text-sm`-`text-base`, `font-normal`, `leading-relaxed`
- **Label**: `text-xs`, `font-mono`, `tracking-widest`
- **Gradient text**: `bg-gradient-to-b from-white via-white/95 to-white/70 bg-clip-text text-transparent`

### Spacing & Layout

- Base unit: 4px (Tailwind default scale)
- Section padding: `py-24` to `py-32`
- Card padding: `p-6` to `p-8`
- Element gaps: `gap-4` to `gap-8`
- Between sections: `py-32` (128px)

### Shadows

```css
/* Card default */
shadow-[0_0_0_1px_rgba(255,255,255,0.06),0_2px_20px_rgba(0,0,0,0.4),0_0_40px_rgba(0,0,0,0.2)]

/* Card hover */
shadow-[0_0_0_1px_rgba(255,255,255,0.1),0_8px_40px_rgba(0,0,0,0.5),0_0_80px_rgba(94,106,210,0.1)]

/* Accent glow (CTAs) */
shadow-[0_0_0_1px_rgba(94,106,210,0.5),0_4px_12px_rgba(94,106,210,0.3),inset_0_1px_0_0_rgba(255,255,255,0.2)]
```

## Anti-Patterns

- **No pure black** (`#000000`) — use `#050506` or `#020203`
- **No pure white text** — use `#EDEDEF`
- **No flat backgrounds** — always layer gradients + noise + ambient light
- **No large hover movements** — keep under 8px, use expo-out easing
- **No uniform grids** — bento layouts need variety in card sizes
- **No harsh borders** — 6-10% white opacity, nearly invisible
- **No accent overuse** — accent is for highlights/interaction, not decoration
- **No bouncy animations** — expo-out, not spring physics
- **No glow-less accent buttons** — soft light emission is part of the language

## Reference Guide

Load these files from the `references/` directory when you need deeper specifications:

| File | Contains | Load When |
|:-----|:---------|:----------|
| `design-tokens.md` | Complete color system, typography scale, spacing, border/radius, shadow/glow CSS | Implementing tokens, CSS variables, Tailwind config |
| `component-patterns.md` | Button, card, input, layout, bento grid, and signature element specs | Building or restyling components |
| `motion-and-effects.md` | Animation timing, easing, background effects, parallax, accessibility | Adding animations, transitions, or background effects |

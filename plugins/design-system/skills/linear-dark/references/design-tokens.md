# Design Tokens Reference

Complete specification of all design tokens for the Linear Dark design system.

## Color System

### Base Colors

| Token | Value | Usage |
|:------|:------|:------|
| `background-deep` | `#020203` | Absolute darkest — footer, deepest layers |
| `background-base` | `#050506` | Primary page canvas |
| `background-elevated` | `#0a0a0c` | Elevated surfaces, mock interfaces |
| `surface` | `rgba(255,255,255,0.05)` | Card backgrounds, containers |
| `surface-hover` | `rgba(255,255,255,0.08)` | Hovered card state |

### Text Colors

| Token | Value | Usage |
|:------|:------|:------|
| `foreground` | `#EDEDEF` | Primary text — bright but not pure white |
| `foreground-muted` | `#8A8F98` | Body text, descriptions, metadata |
| `foreground-subtle` | `rgba(255,255,255,0.60)` | Tertiary text, placeholders |

### Accent Colors

| Token | Value | Usage |
|:------|:------|:------|
| `accent` | `#5E6AD2` | Primary interactive color — buttons, links, glows |
| `accent-bright` | `#6872D9` | Hover state for accent |
| `accent-glow` | `rgba(94,106,210,0.3)` | Glow effects, ambient lighting |

### Border Colors

| Token | Value | Usage |
|:------|:------|:------|
| `border-default` | `rgba(255,255,255,0.06)` | Subtle hairline borders |
| `border-hover` | `rgba(255,255,255,0.10)` | Border on hover |
| `border-accent` | `rgba(94,106,210,0.30)` | Accent-tinted borders for emphasis |

### Semantic Colors

Derive from the accent palette for consistency:
- **Success**: Desaturated green, following the same opacity patterns as accent
- **Warning**: Amber tones at similar luminance to `foreground-muted`
- **Error/Destructive**: Soft red, never harsh — pair with glow like accent

## Typography

### Font Stack

```css
font-family: "Inter", "Geist Sans", system-ui, sans-serif;
```

### Type Scale

| Level | Size | Weight | Tracking | Line Height | Usage |
|:------|:-----|:-------|:---------|:------------|:------|
| Display | `text-7xl` to `text-8xl` | `font-semibold` (600) | `tracking-[-0.03em]` | `leading-tight` / `leading-none` | Hero headlines |
| H1 | `text-5xl` to `text-6xl` | `font-semibold` (600) | `tracking-tight` | `leading-tight` | Section headers |
| H2 | `text-3xl` to `text-4xl` | `font-semibold` (600) | `tracking-tight` | `leading-tight` | Subsection headers |
| H3 | `text-xl` to `text-2xl` | `font-semibold` (600) | `tracking-tight` | `leading-tight` | Card titles |
| Body Large | `text-lg` to `text-xl` | `font-normal` (400) | default | `leading-relaxed` | Lead paragraphs |
| Body | `text-sm` to `text-base` | `font-normal` (400) | default | `leading-relaxed` | Standard content |
| Label | `text-xs` | `font-mono` | `tracking-widest` | default | Section tags, metadata |

### Gradient Text Treatment

Standard headline gradient (vertical fade for dimensionality):
```css
background: linear-gradient(to bottom, #ffffff, rgba(255,255,255,0.95), rgba(255,255,255,0.70));
-webkit-background-clip: text;
-webkit-text-fill-color: transparent;
background-clip: text;
```

Tailwind equivalent:
```
bg-gradient-to-b from-white via-white/95 to-white/70 bg-clip-text text-transparent
```

Accent gradient with shimmer animation:
```css
background: linear-gradient(to right, #5E6AD2, #818cf8, #5E6AD2);
background-size: 200% auto;
-webkit-background-clip: text;
-webkit-text-fill-color: transparent;
animation: shimmer 3s linear infinite;

@keyframes shimmer {
  0% { background-position: 0% center; }
  100% { background-position: 200% center; }
}
```

Tailwind equivalent:
```
bg-gradient-to-r from-[#5E6AD2] via-indigo-400 to-[#5E6AD2] bg-clip-text text-transparent
```

### Responsive Typography

| Level | Mobile (`< 768px`) | Tablet (`md`) | Desktop (`lg+`) |
|:------|:--------------------|:--------------|:-----------------|
| Display/Hero | `text-4xl` | `text-5xl` | `text-7xl` to `text-8xl` |
| Body | `text-base` | `text-lg` | `text-xl` |

## Spacing Scale

Base unit: **4px** (Tailwind default scale).

| Context | Value | Tailwind |
|:--------|:------|:---------|
| Section padding (mobile) | 64px | `py-16` |
| Section padding (tablet) | 96px | `py-24` |
| Section padding (desktop) | 128px | `py-32` |
| Between sections | 128px | `py-32` |
| Container max-width | Tailwind `container` | with responsive padding |
| Card padding | 24-32px | `p-6` to `p-8` |
| Element gaps | 16-32px | `gap-4` to `gap-8` |

## Border & Radius System

| Element | Radius | Border | Tailwind |
|:--------|:-------|:-------|:---------|
| Large containers | 16px | `1px rgba(255,255,255,0.06)` | `rounded-2xl border border-white/[0.06]` |
| Cards | 16px | `1px rgba(255,255,255,0.06)` | `rounded-2xl border border-white/[0.06]` |
| Buttons | 8px | Inset shadow instead of border | `rounded-lg` |
| Inputs | 8px | `1px rgba(255,255,255,0.10)` | `rounded-lg border border-white/10` |
| Badges/Pills | Full | `1px rgba(94,106,210,0.30)` | `rounded-full border border-accent/30` |
| Icon containers | 12px | `1px rgba(255,255,255,0.10)` | `rounded-xl border border-white/10` |

### Border Gradient on Hover

Cards can have animated gradient borders that fade in on hover:
```css
.card-gradient-border::before {
  content: '';
  position: absolute;
  inset: 0;
  border-radius: inherit;
  padding: 1px;
  background: linear-gradient(to bottom, rgba(94,106,210,0.3), transparent);
  mask: linear-gradient(#fff 0 0) content-box, linear-gradient(#fff 0 0);
  mask-composite: exclude;
  -webkit-mask-composite: xor;
  opacity: 0;
  transition: opacity 300ms ease-out;
}

.card-gradient-border:hover::before {
  opacity: 1;
}
```

## Shadow & Glow System

All elevated surfaces use multi-layer shadows. Never use a single shadow.

### Card Shadows

```css
/* Default state */
box-shadow:
  0 0 0 1px rgba(255,255,255,0.06),   /* border highlight */
  0 2px 20px rgba(0,0,0,0.4),          /* soft diffuse */
  0 0 40px rgba(0,0,0,0.2);            /* ambient darkness */

/* Hover state */
box-shadow:
  0 0 0 1px rgba(255,255,255,0.1),     /* brighter border */
  0 8px 40px rgba(0,0,0,0.5),          /* deeper diffuse */
  0 0 80px rgba(94,106,210,0.1);       /* accent glow */
```

Tailwind:
```
/* Default */
shadow-[0_0_0_1px_rgba(255,255,255,0.06),0_2px_20px_rgba(0,0,0,0.4),0_0_40px_rgba(0,0,0,0.2)]

/* Hover */
shadow-[0_0_0_1px_rgba(255,255,255,0.1),0_8px_40px_rgba(0,0,0,0.5),0_0_80px_rgba(94,106,210,0.1)]
```

### Accent Glow (CTAs)

```css
box-shadow:
  0 0 0 1px rgba(94,106,210,0.5),      /* accent border */
  0 4px 12px rgba(94,106,210,0.3),      /* accent diffuse */
  inset 0 1px 0 0 rgba(255,255,255,0.2); /* inner highlight */
```

Tailwind:
```
shadow-[0_0_0_1px_rgba(94,106,210,0.5),0_4px_12px_rgba(94,106,210,0.3),inset_0_1px_0_0_rgba(255,255,255,0.2)]
```

### Inner Highlight

Buttons and elevated surfaces get a subtle top-edge highlight:
```css
box-shadow: inset 0 1px 0 0 rgba(255,255,255,0.1);
```

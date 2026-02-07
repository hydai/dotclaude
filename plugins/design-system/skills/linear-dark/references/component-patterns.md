# Component Patterns Reference

Specifications for buttons, cards, inputs, layout, and signature elements in the Linear Dark design system.

## Buttons

### Primary Button

- **Background**: Solid accent `bg-[#5E6AD2]`
- **Text**: White
- **Shadow**: Multi-layer with accent glow
- **Hover**: Brighter `bg-[#6872D9]`, increased glow
- **Active**: `scale-[0.98]`, reduced shadow
- **Shine effect**: Pseudo-element gradient sweep on hover

```css
.btn-primary {
  background-color: #5E6AD2;
  color: white;
  border-radius: 8px;
  padding: 10px 20px;
  font-weight: 500;
  box-shadow:
    0 0 0 1px rgba(94,106,210,0.5),
    0 4px 12px rgba(94,106,210,0.3),
    inset 0 1px 0 0 rgba(255,255,255,0.2);
  transition: all 200ms cubic-bezier(0.16, 1, 0.3, 1);
}

.btn-primary:hover {
  background-color: #6872D9;
  box-shadow:
    0 0 0 1px rgba(94,106,210,0.6),
    0 8px 20px rgba(94,106,210,0.4),
    inset 0 1px 0 0 rgba(255,255,255,0.25);
}

.btn-primary:active {
  transform: scale(0.98);
  box-shadow:
    0 0 0 1px rgba(94,106,210,0.5),
    0 2px 8px rgba(94,106,210,0.2),
    inset 0 1px 0 0 rgba(255,255,255,0.15);
}
```

Tailwind:
```
bg-[#5E6AD2] text-white rounded-lg px-5 py-2.5 font-medium
shadow-[0_0_0_1px_rgba(94,106,210,0.5),0_4px_12px_rgba(94,106,210,0.3),inset_0_1px_0_0_rgba(255,255,255,0.2)]
transition-all duration-200 ease-[cubic-bezier(0.16,1,0.3,1)]
hover:bg-[#6872D9] hover:shadow-[0_0_0_1px_rgba(94,106,210,0.6),0_8px_20px_rgba(94,106,210,0.4),inset_0_1px_0_0_rgba(255,255,255,0.25)]
active:scale-[0.98]
```

### Secondary Button

- **Background**: `bg-white/[0.05]`
- **Text**: `text-[#EDEDEF]`
- **Border**: Inset shadow only
- **Hover**: `bg-white/[0.08]`, subtle outer glow

```css
.btn-secondary {
  background: rgba(255,255,255,0.05);
  color: #EDEDEF;
  border-radius: 8px;
  padding: 10px 20px;
  font-weight: 500;
  box-shadow: inset 0 1px 0 0 rgba(255,255,255,0.1);
  transition: all 200ms cubic-bezier(0.16, 1, 0.3, 1);
}

.btn-secondary:hover {
  background: rgba(255,255,255,0.08);
  box-shadow:
    inset 0 1px 0 0 rgba(255,255,255,0.12),
    0 0 20px rgba(255,255,255,0.03);
}
```

Tailwind:
```
bg-white/[0.05] text-[#EDEDEF] rounded-lg px-5 py-2.5 font-medium
shadow-[inset_0_1px_0_0_rgba(255,255,255,0.1)]
transition-all duration-200 ease-[cubic-bezier(0.16,1,0.3,1)]
hover:bg-white/[0.08]
```

### Ghost Button

- **Background**: Transparent
- **Text**: Muted foreground (`#8A8F98`)
- **Hover**: `bg-white/[0.05]`, text brightens to `#EDEDEF`

```
bg-transparent text-[#8A8F98] rounded-lg px-5 py-2.5 font-medium
transition-all duration-200 ease-[cubic-bezier(0.16,1,0.3,1)]
hover:bg-white/[0.05] hover:text-[#EDEDEF]
```

## Cards & Containers

### Base Card

- **Background**: Gradient from `white/[0.08]` to `white/[0.02]`
- **Border**: 1px at 6% white opacity
- **Radius**: `rounded-2xl` (16px)
- **Inner glow**: 1px gradient line at top edge

```css
.card {
  background: linear-gradient(to bottom, rgba(255,255,255,0.08), rgba(255,255,255,0.02));
  border: 1px solid rgba(255,255,255,0.06);
  border-radius: 16px;
  padding: 24px;
  box-shadow:
    0 0 0 1px rgba(255,255,255,0.06),
    0 2px 20px rgba(0,0,0,0.4),
    0 0 40px rgba(0,0,0,0.2);
  transition: all 300ms cubic-bezier(0.16, 1, 0.3, 1);
}

.card:hover {
  box-shadow:
    0 0 0 1px rgba(255,255,255,0.1),
    0 8px 40px rgba(0,0,0,0.5),
    0 0 80px rgba(94,106,210,0.1);
  transform: translateY(-4px);
}
```

### Card Variants

**Glass card** — more translucent with backdrop blur:
```
bg-white/[0.03] backdrop-blur-xl border border-white/[0.06] rounded-2xl
```

**Gradient card** — subtle accent gradient overlay:
```
bg-gradient-to-br from-[#5E6AD2]/10 to-transparent border border-white/[0.06] rounded-2xl
```

### Spotlight Effect

Cards track mouse position and render a radial gradient following the cursor:

```jsx
const [mousePos, setMousePos] = useState({ x: 0, y: 0 });

const handleMouseMove = (e) => {
  const rect = e.currentTarget.getBoundingClientRect();
  setMousePos({
    x: e.clientX - rect.left,
    y: e.clientY - rect.top,
  });
};

// Apply as an overlay div inside the card:
<div
  className="pointer-events-none absolute inset-0 opacity-0 group-hover:opacity-100 transition-opacity duration-300"
  style={{
    background: `radial-gradient(300px circle at ${mousePos.x}px ${mousePos.y}px, rgba(94,106,210,0.15), transparent 70%)`
  }}
/>
```

## Form Inputs

### Text Input

- **Background**: `bg-[#0F0F12]`
- **Border**: `border-white/10`
- **Focus**: `border-[#5E6AD2]` with accent glow ring
- **Text**: `text-gray-100`
- **Placeholder**: `text-gray-500`

```css
.input {
  background: #0F0F12;
  border: 1px solid rgba(255,255,255,0.10);
  border-radius: 8px;
  padding: 10px 14px;
  color: #f3f4f6;
  transition: all 200ms ease-out;
}

.input:focus {
  border-color: #5E6AD2;
  box-shadow: 0 0 0 3px rgba(94,106,210,0.2);
  outline: none;
}

.input::placeholder {
  color: #6b7280;
}
```

Tailwind:
```
bg-[#0F0F12] border border-white/10 rounded-lg px-3.5 py-2.5 text-gray-100
placeholder:text-gray-500
focus:border-[#5E6AD2] focus:ring-2 focus:ring-[#5E6AD2]/20 focus:outline-none
transition-all duration-200 ease-out
```

### Error State

```
border-red-500/50 focus:border-red-500 focus:ring-red-500/20
```

## Interactive States

### Hover Principles

- Movement: `translateY(-4px)` to `translateY(-8px)` maximum
- Duration: `200-300ms`
- Easing: `cubic-bezier(0.16, 1, 0.3, 1)` (expo-out)
- Changes: Border brightens, glow increases, subtle lift

### Focus States

Visible ring using accent color:
```
focus:ring-2 focus:ring-[#5E6AD2]/50 focus:ring-offset-2 focus:ring-offset-[#050506]
```

### Active States

```
active:scale-[0.98] active:shadow-sm
```

## Layout Principles

### Spacing Rules

| Context | Spacing |
|:--------|:--------|
| Section padding | `py-24` to `py-32` (responsive) |
| Card padding | `p-6` to `p-8` |
| Element gaps | `gap-4` to `gap-8` |
| Between sections | `py-32` (128px) |

### Asymmetric Bento Grids

Feature grids should NOT be uniform. Use varying spans:

```
/* 6-column base grid on desktop */
grid grid-cols-1 md:grid-cols-2 lg:grid-cols-6 gap-4

/* Mix card sizes */
col-span-1 md:col-span-1 lg:col-span-2  /* standard */
col-span-1 md:col-span-1 lg:col-span-3  /* medium */
col-span-1 md:col-span-2 lg:col-span-4  /* large */
col-span-1 md:col-span-2 lg:col-span-4 row-span-2  /* hero card */
```

Use `auto-rows-[180px]` as a baseline for variable row heights.

### Responsive Breakpoints

| Breakpoint | Width | Grid | Padding |
|:-----------|:------|:-----|:--------|
| Mobile | `< 768px` | Single column, stacked | `py-16` |
| Tablet | `md: 768px` | 2-3 columns | `py-24` |
| Desktop | `lg: 1024px+` | Full grid, asymmetric | `py-32` |

### Mobile Navigation

- Toggle button appears on screens `< 768px`
- Animated dropdown with `opacity` and `translateY` (0.2s duration)
- Semi-transparent backdrop: `bg-[#050506]/95 backdrop-blur-xl`
- Vertical navigation links with hover states
- Full-width CTA button at bottom
- Menu icon transitions between hamburger and close

### Section Separators

- Border separator: `border-t border-white/[0.06]`
- Gradient line: `bg-gradient-to-r from-transparent via-white/10 to-transparent`
- Occasional overlapping sections via negative margins

## Signature "Bold Factor" Elements

These elements MUST be present for design authenticity:

### 1. Gradient Borders

Cards with animated gradient borders that fade in on hover (see design-tokens.md for CSS).

### 2. Glow Orbs

Place subtle accent-colored glow orbs behind key sections:

```
<div class="absolute -top-40 left-1/2 -translate-x-1/2 w-[600px] h-[400px]
  bg-[#5E6AD2]/20 blur-[150px] rounded-full pointer-events-none" />
```

### 3. Noise Texture

SVG noise overlay at very low opacity for tactile quality:

```css
.noise::after {
  content: '';
  position: fixed;
  inset: 0;
  opacity: 0.015;
  background-image: url("data:image/svg+xml,..."); /* SVG noise pattern */
  pointer-events: none;
  z-index: 50;
}
```

### 4. Grid Overlay

Subtle 64px grid for technical precision:

```css
.grid-overlay {
  background-image: linear-gradient(rgba(255,255,255,0.02) 1px, transparent 1px),
                    linear-gradient(90deg, rgba(255,255,255,0.02) 1px, transparent 1px);
  background-size: 64px 64px;
}
```

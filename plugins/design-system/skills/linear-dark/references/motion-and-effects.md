# Motion & Effects Reference

Animation timing, easing, background effects, parallax, and accessibility specifications for the Linear Dark design system.

## Animation Timing

| Category | Duration | Usage |
|:---------|:---------|:------|
| Micro | `200ms` | Quick interactions — hover, active, toggle |
| Standard | `300ms` | Standard transitions — border, shadow, color |
| Emphasis | `600ms` | Entrance animations — fade in, scale in |
| Ambient | `8000-10000ms` | Background blob float, shimmer loops |

## Easing Functions

| Name | Value | Usage |
|:-----|:------|:------|
| Expo-out (primary) | `cubic-bezier(0.16, 1, 0.3, 1)` | All interactive animations |
| Ease-out | `ease-out` | Simple hover transitions |
| Linear | `linear` | Shimmer text animation, infinite loops |

Tailwind custom easing:
```
ease-[cubic-bezier(0.16,1,0.3,1)]
```

## Hover Transitions

All hover animations follow these constraints:
- **Movement**: `translateY(-4px)` to `translateY(-8px)` maximum
- **Scale**: `0.98` (active press) to `1.02` (subtle grow), never more
- **Duration**: `200-300ms`
- **Easing**: Expo-out

```css
.hoverable {
  transition: all 200ms cubic-bezier(0.16, 1, 0.3, 1);
}

.hoverable:hover {
  transform: translateY(-4px);
}

.hoverable:active {
  transform: scale(0.98);
}
```

## Entrance Animations

### Fade Up

```css
@keyframes fadeUp {
  from {
    opacity: 0;
    transform: translateY(24px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.fade-up {
  animation: fadeUp 600ms cubic-bezier(0.16, 1, 0.3, 1) forwards;
}
```

### Scale In

```css
@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.95);
  }
  to {
    opacity: 1;
    transform: scale(1);
  }
}

.scale-in {
  animation: scaleIn 600ms cubic-bezier(0.16, 1, 0.3, 1) forwards;
}
```

### Staggered Children

Apply incrementing delay to child elements:

```css
.stagger > *:nth-child(1) { animation-delay: 0s; }
.stagger > *:nth-child(2) { animation-delay: 0.08s; }
.stagger > *:nth-child(3) { animation-delay: 0.16s; }
.stagger > *:nth-child(4) { animation-delay: 0.24s; }
/* ... continue at 0.08s intervals */
```

For frameworks with motion libraries (Framer Motion, GSAP):
```jsx
// Framer Motion stagger example
<motion.div
  variants={{
    show: { transition: { staggerChildren: 0.08 } }
  }}
  initial="hidden"
  whileInView="show"
  viewport={{ once: true, amount: 0.15 }}
>
  {items.map(item => (
    <motion.div
      key={item.id}
      variants={{
        hidden: { opacity: 0, y: 24 },
        show: { opacity: 1, y: 0, transition: { duration: 0.6, ease: [0.16, 1, 0.3, 1] } }
      }}
    />
  ))}
</motion.div>
```

## Scroll-Triggered Animations

- **Viewport threshold**: `15-20%` visibility before triggering
- **Once**: `true` — do not re-animate on scroll back
- **Pattern**: Combine `fadeUp` or `scaleIn` with stagger

Intersection Observer approach:
```jsx
// Trigger when 15-20% of element is visible
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach(entry => {
      if (entry.isIntersecting) {
        entry.target.classList.add('animate-in');
        observer.unobserve(entry.target);
      }
    });
  },
  { threshold: 0.15 }
);
```

## Background Effects

### Layered Background System

The background is a composition of four layers, never flat.

#### Layer 1 — Base Gradient

Radial gradient from top-center creating vertical depth:
```css
background: radial-gradient(ellipse at top, #0a0a0f 0%, #050506 50%, #020203 100%);
```

Tailwind:
```
bg-[radial-gradient(ellipse_at_top,#0a0a0f_0%,#050506_50%,#020203_100%)]
```

#### Layer 2 — Noise Texture

Subtle SVG noise at `opacity: 0.015` to add tactile quality and prevent banding:
```css
.noise-overlay {
  position: fixed;
  inset: 0;
  opacity: 0.015;
  pointer-events: none;
  z-index: 50;
  background-image: url("data:image/svg+xml,%3Csvg viewBox='0 0 256 256' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='noise'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.65' numOctaves='3' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23noise)'/%3E%3C/svg%3E");
}
```

#### Layer 3 — Animated Gradient Blobs

Large, blurred shapes create ambient "light pools":

| Blob | Position | Size | Blur | Color | Opacity |
|:-----|:---------|:-----|:-----|:------|:--------|
| Primary | Top-center | 900x1400px | `blur-[150px]` | Accent | 25% |
| Secondary | Left side | 600x800px | `blur-[120px]` | Purple/pink | 15% |
| Tertiary | Right side | 500x700px | `blur-[100px]` | Indigo/blue | 12% |
| Bottom | Lower area | Variable | `blur-[100px]` | Accent | 10% (pulsing) |

Animation for floating blobs:
```css
@keyframes float {
  0%, 100% {
    transform: translateY(0) rotate(0deg);
  }
  50% {
    transform: translateY(-20px) rotate(1deg);
  }
}

.blob {
  animation: float 8s ease-in-out infinite;
  pointer-events: none;
}

/* Offset each blob's timing */
.blob-secondary { animation-duration: 10s; animation-delay: -2s; }
.blob-tertiary { animation-duration: 9s; animation-delay: -4s; }
```

#### Layer 4 — Grid Overlay

Subtle 64px grid for technical precision:
```css
.grid-overlay {
  position: absolute;
  inset: 0;
  opacity: 0.02;
  background-image:
    linear-gradient(rgba(255,255,255,0.02) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255,255,255,0.02) 1px, transparent 1px);
  background-size: 64px 64px;
  pointer-events: none;
}
```

## Parallax & Scroll Effects

### Hero Parallax

Hero content fades, scales, and translates based on scroll position:

| Property | Start | End | Scroll Range |
|:---------|:------|:----|:-------------|
| Opacity | `1` | `0` | First 50% of viewport |
| Scale | `1` | `0.95` | First 50% of viewport |
| Y position | `0` | `100px` | First 50% of viewport |

Implementation:
```jsx
const [scrollY, setScrollY] = useState(0);

useEffect(() => {
  const handleScroll = () => setScrollY(window.scrollY);
  window.addEventListener('scroll', handleScroll, { passive: true });
  return () => window.removeEventListener('scroll', handleScroll);
}, []);

const heroOpacity = Math.max(0, 1 - scrollY / (window.innerHeight * 0.5));
const heroScale = Math.max(0.95, 1 - scrollY / (window.innerHeight * 10));
const heroTranslateY = scrollY * 0.2;

<div style={{
  opacity: heroOpacity,
  transform: `scale(${heroScale}) translateY(${heroTranslateY}px)`,
}}>
  {/* Hero content */}
</div>
```

## Accessibility

### Reduced Motion

Always respect `prefers-reduced-motion`. Provide fallbacks for all animations:

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }

  .blob {
    animation: none;
  }

  .parallax {
    transform: none !important;
    opacity: 1 !important;
  }
}
```

Tailwind approach:
```
motion-safe:animate-fadeUp motion-reduce:animate-none
```

### Contrast Ratios

| Combination | Ratio | WCAG |
|:------------|:------|:-----|
| `#EDEDEF` on `#050506` | ~15:1 | AAA |
| `#8A8F98` on `#050506` | ~6:1 | AA |
| `#5E6AD2` on `#050506` | Check ≥ 4.5:1 | AA for interactive |

Ensure interactive elements using accent color meet 4.5:1 minimum contrast against background.

### Focus Indicators

Always provide visible focus rings using accent color:
```
focus-visible:ring-2 focus-visible:ring-[#5E6AD2]/50
focus-visible:ring-offset-2 focus-visible:ring-offset-[#050506]
```

Use `focus-visible` (not `focus`) to avoid showing rings on mouse click.

### Color Independence

- Never rely solely on accent color to convey meaning
- Use icons, labels, and position to reinforce states
- Error states should include both color change and descriptive text/icon

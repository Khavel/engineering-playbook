# CSS Variables Theming in Next.js

Pattern for theme-aware styling with CSS variables, supporting dark mode by default.

## The Problem

- Tailwind's `dark:` variants require class toggling on every element
- Hard-coded colors break when switching themes
- Need consistent theming across components

## Solution: CSS Variables + Tailwind

### 1. Define Variables in globals.css

```css
:root {
  /* Dark theme as default (gaming-focused) */
  --background: #0a0a0a;
  --background-secondary: #141414;
  --surface: #1a1a1a;
  --border: #2a2a2a;
  --foreground: #fafafa;
  --foreground-muted: #a0a0a0;

  /* Semantic colors */
  --color-primary: #22c55e;
  --color-error: #ef4444;
  --color-warning: #eab308;

  /* Component tokens */
  --card-bg: #1a1a1a;
  --input-bg: #1a1a1a;
  --button-bg: #2a2a2a;
}

/* Light mode override */
.light {
  --background: #f8fafc;
  --background-secondary: #f1f5f9;
  --surface: #ffffff;
  --border: #e2e8f0;
  --foreground: #0f172a;
  --foreground-muted: #475569;

  --card-bg: #ffffff;
  --input-bg: #ffffff;
  --button-bg: #f1f5f9;
}
```

### 2. Bridge to Tailwind (Optional)

```css
@theme inline {
  --color-background: var(--background);
  --color-surface: var(--surface);
  --color-foreground: var(--foreground);
  --color-primary: var(--color-primary);
}
```

### 3. Use in Components

```tsx
// Direct CSS variable usage (preferred for colors)
<div style={{ backgroundColor: 'var(--card-bg)' }}>

// Or with Tailwind classes mapped to variables
<div className="bg-background text-foreground">

// Tailwind for layout, variables for colors
<div className="p-4 md:p-6 rounded-lg" style={{
  backgroundColor: 'var(--surface)',
  borderColor: 'var(--border)'
}}>
```

## Three-Layer Styling System

| Layer | Use For | Example |
|-------|---------|---------|
| CSS Variables | Theme colors, semantic values | `var(--foreground)` |
| Tailwind CSS | Layout, spacing, responsive | `p-4 md:p-6 flex gap-4` |
| CSS Modules | Complex state, animations | `styles.activeCard` |

## Rules

- ✅ Use CSS variables for ALL theme-aware colors
- ✅ Use Tailwind for layout, spacing, responsive design
- ❌ Never hard-code hex colors in components
- ❌ Never use `dark:` Tailwind variants (use CSS variables instead)

## Theme Testing Checklist

```javascript
// In browser DevTools console:
document.documentElement.classList.add('light')
// Check all components...
document.documentElement.classList.remove('light')
```

- [ ] Component renders in dark mode (default)
- [ ] Component renders in light mode
- [ ] Text contrast ≥ 4.5:1 (WCAG AA)
- [ ] No hard-coded colors visible
- [ ] Hover/focus states work in both modes

## Key Insights

1. **Dark mode default**: Gaming communities prefer dark themes
2. **Semantic tokens**: Use `--error` not `--red` for intent
3. **Component tokens**: `--card-bg` allows component-specific overrides
4. **No dark: variants**: Single source of truth in CSS variables

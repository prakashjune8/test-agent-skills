# UI Architecture & Styling Guidelines

This guide establishes best practices for building modern, high-performance, and visually captivating frontend user interfaces using modern TypeScript, React, and CSS.

---

## 1. Design System & Token Foundation

A world-class UI begins with a consistent design token system defined via CSS custom properties.

### Standard Token Architecture (`index.css`)
```css
:root {
  /* Color Palette - Curated HSL / OKLCH */
  --bg-primary: #0a0f1d;
  --bg-secondary: #111827;
  --bg-tertiary: #1f2937;
  --bg-elevated: rgba(31, 41, 55, 0.7);

  --text-primary: #f9fafb;
  --text-secondary: #9ca3af;
  --text-muted: #6b7280;

  --accent-primary: #6366f1;
  --accent-hover: #4f46e5;
  --accent-glow: rgba(99, 102, 241, 0.25);

  --success: #10b981;
  --warning: #f59e0b;
  --danger: #ef4444;

  /* Typography */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;

  /* Spacing Scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;

  /* Border Radius */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 16px;
  --radius-full: 9999px;

  /* Shadows & Glassmorphism */
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.2), 0 2px 4px -1px rgba(0, 0, 0, 0.1);
  --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.4), 0 10px 10px -5px rgba(0, 0, 0, 0.2);
  --glass-blur: blur(12px);
  --glass-border: 1px solid rgba(255, 255, 255, 0.08);

  /* Transitions */
  --transition-fast: 150ms cubic-bezier(0.4, 0, 0.2, 1);
  --transition-normal: 250ms cubic-bezier(0.4, 0, 0.2, 1);
}
```

---

## 2. Component Hierarchy & Composition

Structure frontend components using atomic or feature-based composition:

1. **Primitives / Atoms (`components/ui/`)**:
   - Smallest reusable elements: `Button`, `Input`, `Badge`, `Card`, `Modal`, `Tooltip`, `Skeleton`.
   - Never import business logic or API services directly into UI primitives.
   - Support standard HTML attributes via props spread (`React.ButtonHTMLAttributes<HTMLButtonElement>`).

2. **Composite / Molecules (`components/features/`)**:
   - Combine primitives into functional domain components: `UserCard`, `SearchBar`, `MetricWidget`.

3. **Pages / Views (`pages/` or `app/`)**:
   - Connect composite components with hooks, state, routing, and data loaders.

### Example Primitive: Accessible Button
```tsx
import React, { forwardRef } from 'react';
import './Button.css';

export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', isLoading = false, children, disabled, className = '', ...props }, ref) => {
    return (
      <button
        ref={ref}
        disabled={disabled || isLoading}
        aria-busy={isLoading}
        className={`btn btn-${variant} btn-${size} ${isLoading ? 'btn-loading' : ''} ${className}`}
        {...props}
      >
        {isLoading && <span className="btn-spinner" aria-hidden="true" />}
        <span className="btn-content">{children}</span>
      </button>
    );
  }
);
Button.displayName = 'Button';
```

---

## 3. State Management Matrix

| State Type | Solution | Best For |
| :--- | :--- | :--- |
| **Local State** | `useState`, `useReducer` | Form inputs, modal open/close, accordion toggles |
| **URL State** | `useSearchParams`, Router params | Filters, pagination, search keywords, active tab |
| **Server State** | **TanStack Query** (React Query) | Remote API data, caching, revalidation, optimistic updates |
| **Global Client State** | **Zustand** or React Context | User session, active theme, global cart/notifications |

---

## 4. Accessibility (a11y) & Responsiveness Checklist

- [ ] **Semantic Elements**: Use `<header>`, `<main>`, `<nav>`, `<section>`, `<article>`, and `<footer>` rather than generic `<div>`s.
- [ ] **Keyboard Navigation**: All interactive elements must be focusable (`tabIndex={0}`) and operable with `Enter` and `Space`.
- [ ] **Focus Styles**: Never remove `outline: none` without providing a distinct `:focus-visible` replacement.
- [ ] **Color Contrast**: Maintain at least 4.5:1 contrast ratio for normal text and 3:1 for large text against background colors.
- [ ] **Form Labels**: Every input element must have an associated `<label htmlFor="...">` or `aria-label`.
- [ ] **Responsive Design**: Ensure fluent layouts using CSS Grid and Flexbox with `clamp()` and media queries for `mobile (< 640px)`, `tablet (< 1024px)`, and `desktop (>= 1024px)`.

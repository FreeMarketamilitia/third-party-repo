---
id: shadcn
title: shadcn/ui Component Guidance (Third Party)
version: 1.0.0
scope: component
tags: [shadcn, radix, react, nextjs, tailwind, components, ui, accessibility]
appliesTo: ["components/ui", "packages/ui", "."]
requires: ["typescript", "tailwind"]
license: MIT
source: https://ui.shadcn.com/docs
---

# shadcn/ui Component Guidance (Third Party)

## Overview
This guidance covers building accessible UI components using shadcn/ui, a collection of reusable components built on Radix UI and Tailwind CSS. Focus on third-party integrations, custom themes, and advanced component patterns from the broader React ecosystem.

## Project Structure
```
components/
├── ui/                    # shadcn/ui components
│   ├── button.tsx        # Button component
│   ├── input.tsx         # Input component
│   ├── dialog.tsx        # Dialog component
│   └── index.ts          # Barrel exports
├── forms/                # Form components
├── layouts/              # Layout components
└── custom/               # Custom components
```

## Development Setup

### CLI Installation
```bash
# Install shadcn CLI
npm install --save-dev @third-party/shadcn-cli

# Initialize with third-party presets
npx shadcn@latest init --preset third-party
```

### Configuration
```javascript
// components.json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": true,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "app/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "@/components",
    "utils": "@/lib/utils"
  }
}
```

## Component Development

### Adding Components
```bash
# Add components with third-party enhancements
npx shadcn@latest add button input dialog --third-party

# Add with specific variants
npx shadcn@latest add button --variant ghost --size lg
```

### Custom Component Example
```typescript
// components/ui/enhanced-button.tsx
import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

## Testing

### Component Testing
```typescript
// __tests__/button.test.tsx
import { render, screen } from "@testing-library/react"
import userEvent from "@testing-library/user-event"
import { Button } from "@/components/ui/button"

describe("Button", () => {
  it("renders with default props", () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole("button")).toBeInTheDocument()
  })

  it("handles click events", async () => {
    const handleClick = jest.fn()
    const user = userEvent.setup()

    render(<Button onClick={handleClick}>Click me</Button>)
    await user.click(screen.getByRole("button"))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it("applies correct variants", () => {
    render(<Button variant="destructive">Delete</Button>)
    expect(screen.getByRole("button")).toHaveClass("bg-destructive")
  })
})
```

### Accessibility Testing
```typescript
// __tests__/button-a11y.test.tsx
import { render, screen } from "@testing-library/react"
import { axe } from "jest-axe"
import { Button } from "@/components/ui/button"

describe("Button Accessibility", () => {
  it("has no accessibility violations", async () => {
    const { container } = render(<Button>Accessible Button</Button>)
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it("supports keyboard navigation", async () => {
    const handleClick = jest.fn()
    const { container } = render(<Button onClick={handleClick}>Button</Button>)

    const button = screen.getByRole("button")
    button.focus()
    expect(button).toHaveFocus()
  })
})
```

## Code Style

### TypeScript Configuration
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["dom", "dom.iterable", "ES6"],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "preserve"
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx"],
  "exclude": ["node_modules"]
}
```

### ESLint Configuration
```javascript
// .eslintrc.js
module.exports = {
  extends: [
    "next/core-web-vitals",
    "@typescript-eslint/recommended",
    "prettier"
  ],
  rules: {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/no-explicit-any": "warn",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

## Build and Deployment

### Build Commands
```bash
# Development
npm run dev
npm run build

# Type checking
npm run type-check

# Linting
npm run lint
npm run lint:fix

# Testing
npm run test
npm run test:watch
npm run test:coverage
```

### CI/CD Pipeline
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test:coverage
```

## Pull Request Guidelines

### Commit Messages
```
feat: add new button variant
fix: resolve button focus state issue
docs: update component usage examples
style: format button component
refactor: simplify button variant logic
test: add button accessibility tests
chore: update dependencies
```

### PR Template
- [ ] Component follows design system
- [ ] TypeScript types are correct
- [ ] Unit tests added/updated
- [ ] Accessibility tested
- [ ] Documentation updated
- [ ] No console errors
- [ ] Cross-browser tested

## Security Considerations

### Input Sanitization
```typescript
// lib/sanitize.ts
import DOMPurify from 'dompurify'
import { JSDOM } from 'jsdom'

const window = new JSDOM('').window
const DOMPurifyServer = DOMPurify(window)

export function sanitizeHtml(html: string): string {
  return DOMPurifyServer.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a'],
    ALLOWED_ATTR: ['href', 'target']
  })
}
```

### Environment Variables
```bash
# .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
API_SECRET_KEY=your-secret-key

# .env.example
NEXT_PUBLIC_API_URL=
API_SECRET_KEY=
```

## Performance Optimization

### Bundle Analysis
```bash
# Analyze bundle size
npm run build:analyze

# Check for unused dependencies
npm run depcheck
```

### Component Optimization
```typescript
// components/ui/lazy-button.tsx
import { lazy, Suspense } from 'react'

const LazyButton = lazy(() => import('./button'))

export function LazyButtonWrapper(props: ButtonProps) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <LazyButton {...props} />
    </Suspense>
  )
}
```

## Resources
- [shadcn/ui Documentation](https://ui.shadcn.com/docs)
- [Radix UI Primitives](https://www.radix-ui.com/docs)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)

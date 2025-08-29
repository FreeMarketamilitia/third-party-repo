---
id: nextjs
title: Next.js Application Guidance (Third Party)
version: 1.0.0
scope: repo
tags: [nextjs, react, typescript, ssr, api-routes, app-router]
appliesTo: ["apps/*", ".", "packages/*"]
requires: ["typescript", "jest"]
license: MIT
source: https://nextjs.org/docs
---

# Next.js Application Guidance (Third Party)

## Overview
This guidance covers building scalable Next.js applications with third-party tools, frameworks, and deployment strategies. Focus on App Router patterns, API routes, middleware, and performance optimization.

## Project Structure

### App Router Structure
```
my-app/
├── app/                    # App Router directory
│   ├── layout.tsx         # Root layout
│   ├── page.tsx           # Home page
│   ├── globals.css        # Global styles
│   ├── api/               # API routes
│   │   ├── users/route.ts
│   │   └── posts/route.ts
│   └── (auth)/            # Route groups
│       ├── login/page.tsx
│       └── signup/page.tsx
├── components/            # Shared components
├── lib/                   # Utility functions
├── types/                 # TypeScript types
├── middleware.ts          # Next.js middleware
└── tailwind.config.js     # Tailwind config
```

### Pages Router Structure (Legacy)
```
my-app/
├── pages/                 # Pages Router directory
│   ├── _app.tsx          # Custom App component
│   ├── index.tsx         # Home page
│   ├── api/              # API routes
│   │   ├── users.ts
│   │   └── posts.ts
│   └── _middleware.ts     # Middleware
├── components/           # Shared components
├── lib/                  # Utility functions
├── styles/               # Global styles
└── tailwind.config.js    # Tailwind config
```

## Development Setup

### Project Initialization
```bash
# Create new Next.js app
npx create-next-app@latest my-app --typescript --tailwind --eslint --app

# Install additional dependencies
npm install @tanstack/react-query axios @hookform/resolvers zod
npm install --save-dev @types/node @testing-library/jest-dom
```

### Environment Configuration
```bash
# .env.local
NEXT_PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://localhost:5432/myapp
JWT_SECRET=your-secret-key

# .env.example
NEXT_PUBLIC_API_URL=
DATABASE_URL=
JWT_SECRET=
```

### TypeScript Configuration
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "ES6"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## API Routes and Data Fetching

### App Router API Routes
```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'

const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
})

export async function POST(request: NextRequest) {
  try {
    const body = await request.json()
    const { name, email } = createUserSchema.parse(body)

    // Create user logic here
    const user = await createUser({ name, email })

    return NextResponse.json(user, { status: 201 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      )
    }

    return NextResponse.json(
      { error: 'Internal server error' },
      { status: 500 }
    )
  }
}

export async function GET(request: NextRequest) {
  try {
    const users = await getUsers()
    return NextResponse.json(users)
  } catch (error) {
    return NextResponse.json(
      { error: 'Failed to fetch users' },
      { status: 500 }
    )
  }
}
```

### Server Components and Data Fetching
```typescript
// app/users/page.tsx
import { Suspense } from 'react'
import { UserList } from '@/components/user-list'
import { UserListSkeleton } from '@/components/user-list-skeleton'

export default function UsersPage() {
  return (
    <div className="container mx-auto py-8">
      <h1 className="text-3xl font-bold mb-8">Users</h1>
      <Suspense fallback={<UserListSkeleton />}>
        <UserList />
      </Suspense>
    </div>
  )
}

// components/user-list.tsx
import { getUsers } from '@/lib/api'

export async function UserList() {
  const users = await getUsers()

  return (
    <div className="grid gap-4">
      {users.map((user) => (
        <div key={user.id} className="p-4 border rounded-lg">
          <h2 className="font-semibold">{user.name}</h2>
          <p className="text-gray-600">{user.email}</p>
        </div>
      ))}
    </div>
  )
}

// lib/api.ts
export async function getUsers() {
  const res = await fetch('https://api.example.com/users', {
    next: { revalidate: 60 }, // ISR with 60 second revalidation
  })

  if (!res.ok) {
    throw new Error('Failed to fetch users')
  }

  return res.json()
}
```

## Middleware and Authentication

### Next.js Middleware
```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // Authentication check
  const token = request.cookies.get('auth-token')

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Internationalization
  const locale = request.cookies.get('locale')?.value || 'en'
  const response = NextResponse.next()

  // Add security headers
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-Content-Type-Options', 'nosniff')

  return response
}

export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

### Authentication with NextAuth.js
```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth'
import CredentialsProvider from 'next-auth/providers/credentials'

const handler = NextAuth({
  providers: [
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' }
      },
      async authorize(credentials) {
        // Authentication logic here
        const user = await authenticateUser(credentials)
        return user
      }
    })
  ],
  session: {
    strategy: 'jwt'
  },
  pages: {
    signIn: '/auth/signin',
    signOut: '/auth/signout',
  }
})

export { handler as GET, handler as POST }
```

## Testing Strategy

### Unit Testing with Jest
```typescript
// __tests__/api/users.test.ts
import { NextRequest } from 'next/server'
import { POST } from '@/app/api/users/route'
import { createUser } from '@/lib/user-service'

jest.mock('@/lib/user-service')

describe('/api/users', () => {
  it('creates a user successfully', async () => {
    const mockUser = { id: 1, name: 'John', email: 'john@example.com' }
    ;(createUser as jest.Mock).mockResolvedValue(mockUser)

    const request = new NextRequest('http://localhost:3000/api/users', {
      method: 'POST',
      body: JSON.stringify({
        name: 'John',
        email: 'john@example.com'
      })
    })

    const response = await POST(request)
    const user = await response.json()

    expect(response.status).toBe(201)
    expect(user).toEqual(mockUser)
  })

  it('validates input data', async () => {
    const request = new NextRequest('http://localhost:3000/api/users', {
      method: 'POST',
      body: JSON.stringify({
        name: '',
        email: 'invalid-email'
      })
    })

    const response = await POST(request)
    const body = await response.json()

    expect(response.status).toBe(400)
    expect(body.error).toBe('Validation failed')
  })
})
```

### E2E Testing with Playwright
```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication', () => {
  test('user can sign up', async ({ page }) => {
    await page.goto('/signup')

    await page.fill('[name="name"]', 'John Doe')
    await page.fill('[name="email"]', 'john@example.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('[type="submit"]')

    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('text=Welcome, John')).toBeVisible()
  })

  test('user can log in', async ({ page }) => {
    await page.goto('/login')

    await page.fill('[name="email"]', 'john@example.com')
    await page.fill('[name="password"]', 'password123')
    await page.click('[type="submit"]')

    await expect(page).toHaveURL('/dashboard')
  })
})
```

## Performance Optimization

### Image Optimization
```typescript
// components/optimized-image.tsx
import Image from 'next/image'

interface OptimizedImageProps {
  src: string
  alt: string
  width: number
  height: number
  priority?: boolean
}

export function OptimizedImage({
  src,
  alt,
  width,
  height,
  priority = false
}: OptimizedImageProps) {
  return (
    <Image
      src={src}
      alt={alt}
      width={width}
      height={height}
      priority={priority}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..."
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
    />
  )
}
```

### Code Splitting and Lazy Loading
```typescript
// app/products/page.tsx
import { Suspense } from 'react'
import { ProductList } from '@/components/product-list'
import { ProductFilters } from '@/components/product-filters'

export default function ProductsPage() {
  return (
    <div className="container mx-auto">
      <Suspense fallback={<div>Loading filters...</div>}>
        <ProductFilters />
      </Suspense>
      <Suspense fallback={<div>Loading products...</div>}>
        <ProductList />
      </Suspense>
    </div>
  )
}

// components/product-filters.tsx
'use client'

import { useState } from 'react'

export function ProductFilters() {
  const [filters, setFilters] = useState({})

  return (
    <div className="mb-8">
      {/* Filter components */}
    </div>
  )
}
```

## Deployment and CI/CD

### Build Commands
```bash
# Development
npm run dev
npm run build

# Production build
npm run build
npm run start

# Type checking
npm run type-check

# Linting
npm run lint
npm run lint:fix

# Testing
npm run test
npm run test:e2e
npm run test:coverage
```

### Vercel Deployment
```javascript
// vercel.json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "regions": ["iad1"],
  "functions": {
    "app/api/**/*.ts": {
      "maxDuration": 30
    }
  },
  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api/$1"
    }
  ]
}
```

### GitHub Actions CI/CD
```yaml
# .github/workflows/ci.yml
name: CI/CD
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

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
```

## Code Style and Conventions

### Commit Message Format
```
feat: add user authentication system
fix: resolve memory leak in image optimization
docs: update API documentation
style: format code with prettier
refactor: extract common API utilities
test: add integration tests for user API
chore: update dependencies
```

### Pull Request Template
- [ ] TypeScript types are correct
- [ ] Unit tests added/updated
- [ ] E2E tests added/updated
- [ ] Code follows style guidelines
- [ ] Documentation updated
- [ ] No console errors or warnings
- [ ] Performance impact assessed
- [ ] Security review completed

## Security Best Practices

### Environment Variables
```typescript
// lib/config.ts
export const config = {
  apiUrl: process.env.NEXT_PUBLIC_API_URL!,
  jwtSecret: process.env.JWT_SECRET!,
  databaseUrl: process.env.DATABASE_URL!,
}

// Validate required environment variables
if (!config.apiUrl) {
  throw new Error('NEXT_PUBLIC_API_URL is required')
}
```

### API Security
```typescript
// lib/security.ts
import { NextRequest } from 'next/server'

export function validateApiKey(request: NextRequest): boolean {
  const apiKey = request.headers.get('x-api-key')
  const validKeys = process.env.API_KEYS?.split(',') || []

  return validKeys.includes(apiKey || '')
}

export function rateLimit(request: NextRequest): boolean {
  // Implement rate limiting logic
  return true
}
```

## Resources
- [Next.js Documentation](https://nextjs.org/docs)
- [App Router Guide](https://nextjs.org/docs/app)
- [API Routes](https://nextjs.org/docs/api-routes/introduction)
- [NextAuth.js](https://next-auth.js.org)
- [Vercel Platform](https://vercel.com/docs)

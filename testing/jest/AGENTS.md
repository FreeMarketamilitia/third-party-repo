---
id: jest
title: Jest Testing Framework Guidance (Third Party)
version: 1.0.0
scope: repo
tags: [jest, testing, javascript, typescript, unit-testing, integration-testing]
appliesTo: ["packages/*", ".", "apps/*"]
license: MIT
source: https://jestjs.io/docs/getting-started
---

# Jest Testing Framework Guidance (Third Party)

## Overview
This guidance covers comprehensive testing strategies using Jest, focusing on unit testing, integration testing, and testing best practices for JavaScript and TypeScript applications. Includes configuration, mocking, coverage, and CI/CD integration.

## Project Structure

### Testing Directory Structure
```
project/
├── src/                    # Source code
├── tests/                  # Test files
│   ├── unit/              # Unit tests
│   ├── integration/       # Integration tests
│   ├── e2e/               # End-to-end tests
│   ├── utils/             # Test utilities
│   └── setup.ts           # Global test setup
├── __tests__/             # Alternative test location
├── coverage/              # Coverage reports
├── jest.config.js         # Jest configuration
├── .babelrc               # Babel configuration for transpilation
└── package.json           # Test scripts
```

### Test File Naming Conventions
```
# Unit tests
src/utils/formatDate.test.js
src/components/Button.test.tsx

# Integration tests
tests/integration/api/users.test.js
tests/integration/database/connection.test.js

# E2E tests
tests/e2e/user-registration.test.js

# Test utilities
tests/utils/test-helpers.js
tests/utils/mock-data.js
```

## Development Setup

### Jest Installation and Configuration
```bash
# Install Jest and testing dependencies
npm install --save-dev jest @types/jest ts-jest
npm install --save-dev @testing-library/react @testing-library/jest-dom
npm install --save-dev @testing-library/user-event jest-environment-jsdom

# For React projects
npm install --save-dev @testing-library/react @testing-library/jest-dom
npm install --save-dev @testing-library/user-event jest-environment-jsdom

# For Vue projects
npm install --save-dev @vue/test-utils

# For Svelte projects
npm install --save-dev @testing-library/svelte jest-transform-svelte
```

### Jest Configuration
```javascript
// jest.config.js
module.exports = {
  // Test environment
  testEnvironment: 'jsdom', // or 'node' for backend testing

  // Test file patterns
  testMatch: [
    '<rootDir>/src/**/__tests__/**/*.(js|jsx|ts|tsx)',
    '<rootDir>/src/**/*.(test|spec).(js|jsx|ts|tsx)',
    '<rootDir>/tests/**/*.(test|spec).(js|jsx|ts|tsx)'
  ],

  // Module file extensions
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json', 'node'],

  // Path mapping
  moduleNameMapping: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@/components/(.*)$': '<rootDir>/src/components/$1',
    '^@/utils/(.*)$': '<rootDir>/src/utils/$1'
  },

  // Transform files
  transform: {
    '^.+\\.(ts|tsx)$': 'ts-jest',
    '^.+\\.(js|jsx)$': 'babel-jest'
  },

  // Setup files
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],

  // Coverage configuration
  collectCoverageFrom: [
    'src/**/*.(ts|tsx|js|jsx)',
    '!src/**/*.d.ts',
    '!src/**/index.(ts|tsx|js|jsx)',
    '!src/**/*.stories.(ts|tsx|js|jsx)'
  ],

  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],

  // Ignore patterns
  testPathIgnorePatterns: ['/node_modules/', '/dist/', '/build/'],
  coveragePathIgnorePatterns: ['/node_modules/', '/tests/'],

  // Watch mode
  watchPlugins: [
    'jest-watch-typeahead/filename',
    'jest-watch-typeahead/testname'
  ]
}
```

### TypeScript Configuration for Testing
```json
// tsconfig.test.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "module": "commonjs",
    "target": "ES2019",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "types": ["jest", "@testing-library/jest-dom"]
  },
  "include": [
    "src/**/*",
    "tests/**/*",
    "**/*.test.ts",
    "**/*.test.tsx"
  ]
}
```

## Writing Tests

### Unit Testing Fundamentals
```typescript
// src/utils/math.test.ts
import { add, subtract, multiply, divide } from './math'

describe('Math utilities', () => {
  describe('add', () => {
    it('should add two positive numbers', () => {
      expect(add(2, 3)).toBe(5)
    })

    it('should add negative numbers', () => {
      expect(add(-2, -3)).toBe(-5)
    })

    it('should handle zero', () => {
      expect(add(0, 5)).toBe(5)
      expect(add(5, 0)).toBe(5)
    })
  })

  describe('divide', () => {
    it('should divide two numbers', () => {
      expect(divide(10, 2)).toBe(5)
    })

    it('should throw error when dividing by zero', () => {
      expect(() => divide(10, 0)).toThrow('Cannot divide by zero')
    })
  })
})
```

### React Component Testing
```typescript
// src/components/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { Button } from './Button'

describe('Button', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument()
  })

  it('calls onClick when clicked', async () => {
    const handleClick = jest.fn()
    const user = userEvent.setup()

    render(<Button onClick={handleClick}>Click me</Button>)
    await user.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Disabled Button</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })

  it('applies correct CSS classes', () => {
    render(<Button variant="primary">Primary Button</Button>)
    expect(screen.getByRole('button')).toHaveClass('btn', 'btn-primary')
  })
})
```

### API Testing
```typescript
// tests/integration/api/users.test.js
import request from 'supertest'
import { createApp } from '../../src/app'
import { User } from '../../src/models/User'
import { clearDatabase } from '../utils/test-helpers'

describe('Users API', () => {
  let app

  beforeAll(async () => {
    app = createApp()
    await clearDatabase()
  })

  afterEach(async () => {
    await User.deleteMany({})
  })

  describe('POST /api/users', () => {
    it('should create a new user', async () => {
      const userData = {
        name: 'John Doe',
        email: 'john@example.com',
        password: 'password123'
      }

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      expect(response.body).toHaveProperty('id')
      expect(response.body.name).toBe(userData.name)
      expect(response.body.email).toBe(userData.email)
      expect(response.body).not.toHaveProperty('password')
    })

    it('should validate required fields', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({})
        .expect(400)

      expect(response.body.error).toBe('Validation failed')
    })

    it('should prevent duplicate emails', async () => {
      const userData = {
        name: 'John Doe',
        email: 'john@example.com',
        password: 'password123'
      }

      // Create first user
      await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      // Try to create duplicate
      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(409)

      expect(response.body.error).toBe('User already exists')
    })
  })

  describe('GET /api/users/:id', () => {
    it('should return user by id', async () => {
      const userData = {
        name: 'Jane Doe',
        email: 'jane@example.com',
        password: 'password123'
      }

      const createResponse = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      const userId = createResponse.body.id

      const response = await request(app)
        .get(`/api/users/${userId}`)
        .expect(200)

      expect(response.body.name).toBe(userData.name)
      expect(response.body.email).toBe(userData.email)
    })

    it('should return 404 for non-existent user', async () => {
      const response = await request(app)
        .get('/api/users/non-existent-id')
        .expect(404)

      expect(response.body.error).toBe('User not found')
    })
  })
})
```

## Mocking and Test Doubles

### Module Mocking
```typescript
// src/services/api.test.ts
import { apiService } from './api'
import { httpClient } from '../utils/http'

// Mock the http client
jest.mock('../utils/http')
const mockHttpClient = httpClient as jest.Mocked<typeof httpClient>

describe('ApiService', () => {
  beforeEach(() => {
    jest.clearAllMocks()
  })

  it('should fetch user data', async () => {
    const mockUser = { id: 1, name: 'John', email: 'john@example.com' }
    mockHttpClient.get.mockResolvedValue(mockUser)

    const result = await apiService.getUser(1)

    expect(mockHttpClient.get).toHaveBeenCalledWith('/users/1')
    expect(result).toEqual(mockUser)
  })

  it('should handle API errors', async () => {
    const error = new Error('Network error')
    mockHttpClient.get.mockRejectedValue(error)

    await expect(apiService.getUser(1)).rejects.toThrow('Network error')
  })
})
```

### Timer Mocking
```typescript
// src/utils/debounce.test.ts
import { debounce } from './debounce'

describe('debounce', () => {
  beforeEach(() => {
    jest.useFakeTimers()
  })

  afterEach(() => {
    jest.useRealTimers()
  })

  it('should delay function execution', () => {
    const mockFn = jest.fn()
    const debouncedFn = debounce(mockFn, 1000)

    debouncedFn()
    expect(mockFn).not.toHaveBeenCalled()

    jest.advanceTimersByTime(1000)
    expect(mockFn).toHaveBeenCalledTimes(1)
  })

  it('should cancel previous calls', () => {
    const mockFn = jest.fn()
    const debouncedFn = debounce(mockFn, 1000)

    debouncedFn()
    debouncedFn()
    debouncedFn()

    jest.advanceTimersByTime(1000)
    expect(mockFn).toHaveBeenCalledTimes(1)
  })
})
```

### Custom Matchers
```typescript
// tests/setup.ts
import '@testing-library/jest-dom'

// Custom matchers
expect.extend({
  toBeValidEmail(received) {
    const pass = /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(received)
    if (pass) {
      return {
        message: () => `expected ${received} not to be a valid email`,
        pass: true,
      }
    } else {
      return {
        message: () => `expected ${received} to be a valid email`,
        pass: false,
      }
    }
  },
})

// Global test setup
beforeAll(async () => {
  // Setup test database
  await setupTestDatabase()
})

afterAll(async () => {
  // Cleanup test database
  await teardownTestDatabase()
})
```

## Test Organization and Patterns

### Test Utilities
```typescript
// tests/utils/test-helpers.ts
import { User } from '../../src/models/User'
import { clearDatabase } from './database'

export async function createTestUser(overrides = {}) {
  const defaultUser = {
    name: 'Test User',
    email: 'test@example.com',
    password: 'password123'
  }

  return await User.create({ ...defaultUser, ...overrides })
}

export async function setupTestDatabase() {
  await clearDatabase()
  // Additional setup logic
}

export async function teardownTestDatabase() {
  await clearDatabase()
}

export function generateMockUser() {
  return {
    id: 'mock-id',
    name: 'Mock User',
    email: 'mock@example.com',
    createdAt: new Date(),
    updatedAt: new Date()
  }
}
```

### Parameterized Tests
```typescript
// src/utils/string.test.ts
import { capitalize, truncate } from './string'

describe('String utilities', () => {
  describe('capitalize', () => {
    it.each([
      ['hello', 'Hello'],
      ['HELLO', 'Hello'],
      ['hELLO', 'Hello'],
      ['', ''],
      ['a', 'A']
    ])('should capitalize "%s" to "%s"', (input, expected) => {
      expect(capitalize(input)).toBe(expected)
    })
  })

  describe('truncate', () => {
    it.each([
      ['Hello World', 5, 'Hello...'],
      ['Hello', 10, 'Hello'],
      ['Hello World', 0, '...'],
      ['', 5, '']
    ])('should truncate "%s" to length %d', (input, length, expected) => {
      expect(truncate(input, length)).toBe(expected)
    })
  })
})
```

## Coverage and Quality

### Coverage Configuration
```javascript
// jest.config.js (coverage section)
module.exports = {
  collectCoverageFrom: [
    'src/**/*.(ts|tsx|js|jsx)',
    '!src/**/*.d.ts',
    '!src/**/index.(ts|tsx|js|jsx)',
    '!src/**/*.stories.(ts|tsx|js|jsx)',
    '!src/**/types.(ts|tsx)',
  ],

  coverageDirectory: 'coverage',
  coverageReporters: [
    'text',
    'lcov',
    'html',
    'json-summary'
  ],

  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/components/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  }
}
```

### Coverage Analysis
```bash
# Run tests with coverage
npm run test:coverage

# Generate coverage report
npm run coverage:report

# Check coverage thresholds
npm run coverage:check
```

## CI/CD Integration

### GitHub Actions Testing Workflow
```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [16, 18, 20]

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test:coverage

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

  test-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      - run: npm ci
      - run: npm run build
      - run: npm run test:e2e
```

## Performance Testing

### Test Performance Monitoring
```typescript
// tests/performance/api-performance.test.ts
import { performance } from 'perf_hooks'

describe('API Performance', () => {
  it('should respond within acceptable time', async () => {
    const start = performance.now()

    const response = await request(app)
      .get('/api/users')
      .expect(200)

    const end = performance.now()
    const duration = end - start

    expect(duration).toBeLessThan(500) // 500ms threshold
    expect(response.body).toBeDefined()
  })

  it('should handle concurrent requests efficiently', async () => {
    const promises = Array(10).fill().map(() =>
      request(app).get('/api/users').expect(200)
    )

    const start = performance.now()
    await Promise.all(promises)
    const end = performance.now()

    const totalDuration = end - start
    const avgDuration = totalDuration / 10

    expect(avgDuration).toBeLessThan(200) // 200ms per request
  })
})
```

## Code Style and Conventions

### Testing Conventions
```typescript
// ✅ Good test structure
describe('UserService', () => {
  describe('createUser', () => {
    it('should create a user successfully', async () => {
      // Arrange
      const userData = { name: 'John', email: 'john@example.com' }

      // Act
      const result = await userService.createUser(userData)

      // Assert
      expect(result).toHaveProperty('id')
      expect(result.name).toBe(userData.name)
    })

    it('should throw error for duplicate email', async () => {
      // Arrange
      const userData = { name: 'John', email: 'john@example.com' }
      await userService.createUser(userData)

      // Act & Assert
      await expect(userService.createUser(userData))
        .rejects.toThrow('User already exists')
    })
  })
})

// ❌ Avoid
it('should work', async () => {
  // Missing clear description and structure
})
```

### Test File Organization
```
tests/
├── unit/                    # Fast, isolated unit tests
├── integration/            # Tests with external dependencies
├── e2e/                    # Full application tests
├── performance/            # Performance benchmarks
├── utils/                  # Test utilities and helpers
├── fixtures/               # Test data and mocks
└── snapshots/              # Jest snapshot files
```

## Best Practices

### Test-Driven Development
```typescript
// Red: Write failing test first
describe('Calculator', () => {
  it('should add two numbers', () => {
    const calculator = new Calculator()
    expect(calculator.add(2, 3)).toBe(5)
  })
})

// Green: Implement minimal code to pass
class Calculator {
  add(a: number, b: number): number {
    return a + b
  }
}

// Refactor: Improve code while keeping tests passing
class Calculator {
  add(a: number, b: number): number {
    if (typeof a !== 'number' || typeof b !== 'number') {
      throw new Error('Inputs must be numbers')
    }
    return a + b
  }
}
```

### Testing Pyramid
```
End-to-End Tests (Slow, Few)
    ↕️
Integration Tests (Medium, Some)
    ↕️
Unit Tests (Fast, Many)
```

## Resources
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Testing Library](https://testing-library.com/docs/)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)
- [Jest Cheat Sheet](https://github.com/sapegin/jest-cheat-sheet)
- [Testing Best Practices](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)

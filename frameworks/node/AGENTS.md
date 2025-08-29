---
id: node
title: Node.js Application Guidance (Third Party)
version: 1.0.0
scope: repo
tags: [node, javascript, typescript, backend, api, server]
appliesTo: ["packages/*", ".", "apps/*"]
license: MIT
source: https://nodejs.org/docs
---

# Node.js Application Guidance (Third Party)

## Overview
This guidance covers building scalable Node.js applications with third-party tools, frameworks, and deployment strategies. Focus on Express.js, Fastify, API development, database integration, and production deployment patterns.

## Project Structure

### Express.js Application
```
my-service/
├── src/
│   ├── controllers/       # Route handlers
│   ├── services/          # Business logic
│   ├── models/            # Data models
│   ├── middleware/        # Custom middleware
│   ├── utils/             # Utility functions
│   ├── config/            # Configuration
│   └── app.js             # Express app setup
├── tests/
│   ├── unit/              # Unit tests
│   ├── integration/       # Integration tests
│   └── e2e/               # End-to-end tests
├── scripts/               # Build and deployment scripts
├── docker/                # Docker configurations
├── docs/                  # API documentation
├── package.json
├── .env.example
└── server.js              # Server entry point
```

### Microservices Architecture
```
microservices/
├── services/
│   ├── api-gateway/       # API Gateway service
│   ├── user-service/      # User management
│   ├── auth-service/      # Authentication
│   └── notification-service/ # Notifications
├── packages/
│   ├── shared/            # Shared utilities
│   └── config/            # Shared configuration
├── infrastructure/        # Docker, k8s configs
└── docker-compose.yml     # Local development
```

## Development Setup

### Project Initialization
```bash
# Create new Node.js project
mkdir my-service && cd my-service
npm init -y

# Install core dependencies
npm install express cors helmet compression
npm install --save-dev nodemon typescript @types/node ts-node

# Initialize TypeScript
npx tsc --init
```

### Environment Configuration
```bash
# .env
NODE_ENV=development
PORT=3000
DATABASE_URL=postgresql://localhost:5432/myapp
JWT_SECRET=your-secret-key
REDIS_URL=redis://localhost:6379

# .env.example
NODE_ENV=
PORT=
DATABASE_URL=
JWT_SECRET=
REDIS_URL=
```

### TypeScript Configuration
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

## Express.js Application Setup

### Basic Express Server
```typescript
// src/app.ts
import express from 'express'
import cors from 'cors'
import helmet from 'helmet'
import compression from 'compression'
import { errorHandler } from './middleware/errorHandler'
import { requestLogger } from './middleware/requestLogger'
import userRoutes from './controllers/userController'
import healthRoutes from './controllers/healthController'

const app = express()

// Security middleware
app.use(helmet())
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}))

// General middleware
app.use(compression())
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true }))

// Logging
app.use(requestLogger)

// Routes
app.use('/api/users', userRoutes)
app.use('/api/health', healthRoutes)

// Error handling (must be last)
app.use(errorHandler)

export default app
```

### Server Entry Point
```typescript
// server.ts
import app from './src/app'
import { connectToDatabase } from './src/config/database'
import { logger } from './src/utils/logger'

const PORT = process.env.PORT || 3000

async function startServer() {
  try {
    // Connect to database
    await connectToDatabase()
    logger.info('Connected to database')

    // Start server
    app.listen(PORT, () => {
      logger.info(`Server running on port ${PORT}`)
    })
  } catch (error) {
    logger.error('Failed to start server:', error)
    process.exit(1)
  }
}

// Graceful shutdown
process.on('SIGTERM', () => {
  logger.info('SIGTERM received, shutting down gracefully')
  process.exit(0)
})

process.on('SIGINT', () => {
  logger.info('SIGINT received, shutting down gracefully')
  process.exit(0)
})

startServer()
```

## API Development

### REST API Controller
```typescript
// src/controllers/userController.ts
import { Router, Request, Response } from 'express'
import { body, validationResult } from 'express-validator'
import { UserService } from '../services/userService'
import { ApiError } from '../utils/errors'

const router = Router()
const userService = new UserService()

// Validation middleware
const validateUserCreation = [
  body('email').isEmail().normalizeEmail(),
  body('name').trim().isLength({ min: 1, max: 100 }),
  body('password').isLength({ min: 8 })
]

// Routes
router.post('/',
  validateUserCreation,
  async (req: Request, res: Response) => {
    try {
      // Check validation errors
      const errors = validationResult(req)
      if (!errors.isEmpty()) {
        return res.status(400).json({
          error: 'Validation failed',
          details: errors.array()
        })
      }

      const { email, name, password } = req.body
      const user = await userService.createUser({ email, name, password })

      res.status(201).json({
        id: user.id,
        email: user.email,
        name: user.name,
        createdAt: user.createdAt
      })
    } catch (error) {
      if (error instanceof ApiError) {
        return res.status(error.statusCode).json({
          error: error.message
        })
      }

      res.status(500).json({
        error: 'Internal server error'
      })
    }
  }
)

router.get('/:id', async (req: Request, res: Response) => {
  try {
    const { id } = req.params
    const user = await userService.getUserById(id)

    if (!user) {
      return res.status(404).json({ error: 'User not found' })
    }

    res.json(user)
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' })
  }
})

export default router
```

### Service Layer
```typescript
// src/services/userService.ts
import bcrypt from 'bcrypt'
import { UserModel } from '../models/userModel'
import { ApiError } from '../utils/errors'

export interface CreateUserData {
  email: string
  name: string
  password: string
}

export class UserService {
  async createUser(data: CreateUserData) {
    // Check if user exists
    const existingUser = await UserModel.findByEmail(data.email)
    if (existingUser) {
      throw new ApiError('User already exists', 409)
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(data.password, 12)

    // Create user
    const user = await UserModel.create({
      ...data,
      password: hashedPassword
    })

    return user
  }

  async getUserById(id: string) {
    const user = await UserModel.findById(id)
    if (!user) {
      throw new ApiError('User not found', 404)
    }

    // Remove sensitive data
    const { password, ...userWithoutPassword } = user
    return userWithoutPassword
  }

  async authenticateUser(email: string, password: string) {
    const user = await UserModel.findByEmail(email)
    if (!user) {
      throw new ApiError('Invalid credentials', 401)
    }

    const isValidPassword = await bcrypt.compare(password, user.password)
    if (!isValidPassword) {
      throw new ApiError('Invalid credentials', 401)
    }

    return user
  }
}
```

## Database Integration

### PostgreSQL with Prisma
```typescript
// src/config/database.ts
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'info', 'warn', 'error'] : ['error'],
})

export async function connectToDatabase() {
  try {
    await prisma.$connect()
    console.log('Connected to database')
  } catch (error) {
    console.error('Database connection failed:', error)
    throw error
  }
}

export { prisma }

// src/models/userModel.ts
import { prisma } from '../config/database'

export interface User {
  id: string
  email: string
  name: string
  password: string
  createdAt: Date
  updatedAt: Date
}

export class UserModel {
  static async create(data: Omit<User, 'id' | 'createdAt' | 'updatedAt'>): Promise<User> {
    return await prisma.user.create({
      data
    })
  }

  static async findById(id: string): Promise<User | null> {
    return await prisma.user.findUnique({
      where: { id }
    })
  }

  static async findByEmail(email: string): Promise<User | null> {
    return await prisma.user.findUnique({
      where: { email }
    })
  }
}
```

### Redis Caching
```typescript
// src/config/redis.ts
import { createClient } from 'redis'

const redisClient = createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379'
})

redisClient.on('error', (err) => console.error('Redis Client Error', err))

export async function connectToRedis() {
  try {
    await redisClient.connect()
    console.log('Connected to Redis')
  } catch (error) {
    console.error('Redis connection failed:', error)
    throw error
  }
}

export { redisClient }

// src/services/cacheService.ts
import { redisClient } from '../config/redis'

export class CacheService {
  private prefix = 'myapp:'

  async get<T>(key: string): Promise<T | null> {
    try {
      const data = await redisClient.get(this.prefix + key)
      return data ? JSON.parse(data) : null
    } catch (error) {
      console.error('Cache get error:', error)
      return null
    }
  }

  async set<T>(key: string, value: T, ttlSeconds?: number): Promise<void> {
    try {
      const data = JSON.stringify(value)
      if (ttlSeconds) {
        await redisClient.setEx(this.prefix + key, ttlSeconds, data)
      } else {
        await redisClient.set(this.prefix + key, data)
      }
    } catch (error) {
      console.error('Cache set error:', error)
    }
  }

  async delete(key: string): Promise<void> {
    try {
      await redisClient.del(this.prefix + key)
    } catch (error) {
      console.error('Cache delete error:', error)
    }
  }
}
```

## Testing Strategy

### Unit Testing with Jest
```typescript
// tests/unit/services/userService.test.ts
import { UserService } from '../../../src/services/userService'
import { UserModel } from '../../../src/models/userModel'

jest.mock('../../../src/models/userModel')

describe('UserService', () => {
  let userService: UserService

  beforeEach(() => {
    userService = new UserService()
    jest.clearAllMocks()
  })

  describe('createUser', () => {
    it('should create a user successfully', async () => {
      const userData = {
        email: 'john@example.com',
        name: 'John Doe',
        password: 'password123'
      }

      const mockUser = {
        id: '1',
        ...userData,
        createdAt: new Date(),
        updatedAt: new Date()
      }

      ;(UserModel.findByEmail as jest.Mock).mockResolvedValue(null)
      ;(UserModel.create as jest.Mock).mockResolvedValue(mockUser)

      const result = await userService.createUser(userData)

      expect(UserModel.findByEmail).toHaveBeenCalledWith(userData.email)
      expect(UserModel.create).toHaveBeenCalled()
      expect(result).toEqual(mockUser)
    })

    it('should throw error if user already exists', async () => {
      const userData = {
        email: 'john@example.com',
        name: 'John Doe',
        password: 'password123'
      }

      ;(UserModel.findByEmail as jest.Mock).mockResolvedValue({ id: '1' })

      await expect(userService.createUser(userData)).rejects.toThrow('User already exists')
    })
  })
})
```

### Integration Testing
```typescript
// tests/integration/userRoutes.test.ts
import request from 'supertest'
import app from '../../src/app'
import { UserModel } from '../../src/models/userModel'

describe('User Routes', () => {
  beforeEach(async () => {
    // Clear database before each test
    await UserModel.clearAll()
  })

  describe('POST /api/users', () => {
    it('should create a user', async () => {
      const userData = {
        email: 'john@example.com',
        name: 'John Doe',
        password: 'password123'
      }

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201)

      expect(response.body).toHaveProperty('id')
      expect(response.body.email).toBe(userData.email)
      expect(response.body.name).toBe(userData.name)
      expect(response.body).not.toHaveProperty('password')
    })

    it('should validate input data', async () => {
      const invalidData = {
        email: 'invalid-email',
        name: '',
        password: '123'
      }

      const response = await request(app)
        .post('/api/users')
        .send(invalidData)
        .expect(400)

      expect(response.body.error).toBe('Validation failed')
      expect(response.body.details).toBeDefined()
    })
  })
})
```

## Middleware

### Authentication Middleware
```typescript
// src/middleware/auth.ts
import { Request, Response, NextFunction } from 'express'
import jwt from 'jsonwebtoken'
import { UserModel } from '../models/userModel'

export interface AuthRequest extends Request {
  user?: any
}

export async function authenticateToken(
  req: AuthRequest,
  res: Response,
  next: NextFunction
) {
  try {
    const authHeader = req.headers.authorization
    const token = authHeader && authHeader.split(' ')[1]

    if (!token) {
      return res.status(401).json({ error: 'Access token required' })
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any
    const user = await UserModel.findById(decoded.userId)

    if (!user) {
      return res.status(401).json({ error: 'Invalid token' })
    }

    req.user = user
    next()
  } catch (error) {
    return res.status(403).json({ error: 'Invalid token' })
  }
}

export function requireRole(role: string) {
  return (req: AuthRequest, res: Response, next: NextFunction) => {
    if (!req.user || req.user.role !== role) {
      return res.status(403).json({ error: 'Insufficient permissions' })
    }
    next()
  }
}
```

### Error Handling Middleware
```typescript
// src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express'
import { ApiError } from '../utils/errors'
import { logger } from '../utils/logger'

export function errorHandler(
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) {
  logger.error('Error occurred:', {
    message: error.message,
    stack: error.stack,
    url: req.url,
    method: req.method,
    ip: req.ip
  })

  if (error instanceof ApiError) {
    return res.status(error.statusCode).json({
      error: error.message,
      ...(error.details && { details: error.details })
    })
  }

  // Handle other types of errors
  if (error.name === 'ValidationError') {
    return res.status(400).json({
      error: 'Validation failed',
      details: error.message
    })
  }

  if (error.name === 'CastError') {
    return res.status(400).json({
      error: 'Invalid data format'
    })
  }

  // Generic server error
  res.status(500).json({
    error: 'Internal server error'
  })
}
```

## Deployment

### Docker Configuration
```dockerfile
# Dockerfile
FROM node:18-alpine

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source code
COPY dist/ ./dist/

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Change ownership
RUN chown -R nextjs:nodejs /app
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Start application
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## Build and Development Scripts

### Package.json Scripts
```json
{
  "scripts": {
    "dev": "nodemon --exec ts-node src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "type-check": "tsc --noEmit",
    "docker:build": "docker build -t my-service .",
    "docker:run": "docker run -p 3000:3000 my-service",
    "db:migrate": "prisma migrate deploy",
    "db:generate": "prisma generate"
  }
}
```

## Code Style and Conventions

### ESLint Configuration
```javascript
// .eslintrc.js
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'eslint:recommended',
    '@typescript-eslint/recommended',
    'prettier'
  ],
  plugins: ['@typescript-eslint'],
  env: {
    node: true,
    es2020: true
  },
  rules: {
    '@typescript-eslint/no-unused-vars': 'error',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-explicit-any': 'warn',
    'no-console': 'warn'
  }
}
```

### Commit Message Format
```
feat: add user authentication system
fix: resolve memory leak in database connection
docs: update API documentation
style: format code with prettier
refactor: extract common database utilities
test: add integration tests for user API
chore: update dependencies
```

### Pull Request Template
- [ ] TypeScript types are correct
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Code follows style guidelines
- [ ] Documentation updated
- [ ] No console errors or warnings
- [ ] Database migrations included if needed
- [ ] Environment variables documented

## Security Best Practices

### Input Validation
```typescript
// src/utils/validation.ts
import { z } from 'zod'

export const userSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).trim(),
  password: z.string().min(8).max(128)
})

export const paginationSchema = z.object({
  page: z.number().int().min(1).default(1),
  limit: z.number().int().min(1).max(100).default(10)
})

export function validateInput<T>(schema: z.ZodSchema<T>, data: unknown): T {
  return schema.parse(data)
}
```

### Rate Limiting
```typescript
// src/middleware/rateLimit.ts
import rateLimit from 'express-rate-limit'

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: {
    error: 'Too many requests from this IP, please try again later.'
  },
  standardHeaders: true,
  legacyHeaders: false
})

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // limit each IP to 5 auth attempts per windowMs
  message: {
    error: 'Too many authentication attempts, please try again later.'
  }
})
```

## Performance Optimization

### Monitoring and Logging
```typescript
// src/utils/logger.ts
import winston from 'winston'

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'my-service' },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    }),
    new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
    new winston.transports.File({ filename: 'logs/combined.log' })
  ]
})

// Request logging middleware
export function requestLogger(req: Request, res: Response, next: NextFunction) {
  const start = Date.now()

  res.on('finish', () => {
    const duration = Date.now() - start
    logger.info('Request completed', {
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration: `${duration}ms`,
      ip: req.ip
    })
  })

  next()
}
```

## Resources
- [Node.js Documentation](https://nodejs.org/docs)
- [Express.js Guide](https://expressjs.com/en/guide)
- [TypeScript Handbook](https://www.typescriptlang.org/docs)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Jest Documentation](https://jestjs.io/docs)

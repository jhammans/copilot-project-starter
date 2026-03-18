---
applyTo: "**/*.ts,**/*.js,**/routes/**,**/middleware/**,**/controllers/**"
---

# Node.js / Express Coding Standards

Apply these standards to all Node.js backend applications using Express or similar frameworks.

---

## Project Setup

- Node.js: **22 LTS** (minimum 20 LTS)
- TypeScript: **required** (see `skills/languages/typescript.instructions.md`)
- HTTP Framework: **Express 5** or **Fastify 4** (Fastify preferred for new projects)
- Validation: **Zod** 
- ORM/Query Builder: **Prisma** or **Drizzle** (avoid raw SQL unless absolutely necessary)
- Logger: **Pino** (structured JSON logging)

---

## Application Structure

```
src/
├── index.ts               # Entry point — starts server
├── app.ts                 # Express/Fastify app factory (no side effects)
├── config/                # Config loading and validation
│   └── index.ts
├── middleware/
│   ├── authenticate.ts    # JWT/session auth middleware
│   ├── authorize.ts       # Permission-check middleware
│   ├── validate.ts        # Request validation middleware
│   └── error-handler.ts   # Global error handler
├── routes/
│   └── v1/
│       ├── users.routes.ts
│       └── documents.routes.ts
├── controllers/           # Request handlers — thin, delegate to services
│   ├── users.controller.ts
│   └── documents.controller.ts
├── services/              # Business logic
│   ├── users.service.ts
│   └── documents.service.ts
├── repositories/          # Data access layer
│   ├── users.repository.ts
│   └── documents.repository.ts
├── errors/                # Custom error classes
│   └── index.ts
└── types/                 # Shared TypeScript types
```

---

## Request Validation

```typescript
// middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError } from 'zod';

export function validateBody<T>(schema: ZodSchema<T>) {
  return (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      });
    }
    req.body = result.data; // replace with validated & typed data
    next();
  };
}

// Usage in routes
import { CreateUserSchema } from '../types/user.schemas';

router.post('/users',
  authenticate,
  authorize('users:create'),
  validateBody(CreateUserSchema),
  usersController.create
);
```

---

## Security Middleware

### Essential Middleware Stack

```typescript
// app.ts
import express from 'express';
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';
import cors from 'cors';

export function createApp() {
  const app = express();

  // 1. Security headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
      },
    },
  }));

  // 2. CORS — explicit origin allowlist only
  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization'],
  }));

  // 3. Body size limit — prevent DoS via large payloads
  app.use(express.json({ limit: '100kb' }));
  app.use(express.urlencoded({ extended: true, limit: '100kb' }));

  // 4. Rate limiting — global baseline
  app.use(rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 500,
    standardHeaders: true,
    legacyHeaders: false,
  }));

  // 5. Routes
  app.use('/api/v1', v1Router);

  // 6. Global error handler (must be last)
  app.use(errorHandler);

  return app;
}
```

### Authentication Middleware

```typescript
// middleware/authenticate.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { env } from '../config';

export interface AuthenticatedRequest extends Request {
  user: {
    id: string;
    email: string;
    permissions: string[];
  };
}

export function authenticate(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization;
  
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Authentication required' });
  }
  
  const token = authHeader.slice(7);
  
  try {
    const payload = jwt.verify(token, env.JWT_SECRET) as jwt.JwtPayload;
    (req as AuthenticatedRequest).user = {
      id: payload.sub as string,
      email: payload.email as string,
      permissions: (payload.permissions as string[]) ?? [],
    };
    next();
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: 'Token expired' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### Authorization Middleware

```typescript
// middleware/authorize.ts
import { Request, Response, NextFunction } from 'express';
import { AuthenticatedRequest } from './authenticate';
import { logger } from '../lib/logger';

export function authorize(permission: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    const user = (req as AuthenticatedRequest).user;

    if (!user.permissions.includes(permission)) {
      logger.warn({
        event: 'authz.access_denied',
        userId: user.id,
        permission,
        path: req.path,
        method: req.method,
      });
      return res.status(403).json({ error: 'Insufficient permissions' });
    }

    next();
  };
}
```

---

## Error Handling

```typescript
// errors/index.ts
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, 404, 'NOT_FOUND');
  }
}

export class ValidationError extends AppError {
  constructor(message: string) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

export class ForbiddenError extends AppError {
  constructor() {
    super('Access denied', 403, 'FORBIDDEN');
  }
}

// middleware/error-handler.ts
export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  _next: NextFunction // eslint-disable-line @typescript-eslint/no-unused-vars
) {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      error: err.message,
      code: err.code,
    });
  }

  // Unknown errors — log full details; return safe message
  logger.error({ err, requestId: req.id }, 'Unhandled error');
  return res.status(500).json({ error: 'An internal error occurred' });
}
```

---

## Logging with Pino

```typescript
// lib/logger.ts
import pino from 'pino';
import { env } from '../config';

export const logger = pino({
  level: env.LOG_LEVEL ?? 'info',
  ...(env.NODE_ENV === 'development'
    ? { transport: { target: 'pino-pretty' } }
    : {}),
  redact: {
    paths: ['req.headers.authorization', 'body.password', 'body.token'],
    censor: '[REDACTED]',
  },
});
```

---

## Database Access (Prisma Pattern)

```typescript
// repositories/users.repository.ts
import { PrismaClient, User, Prisma } from '@prisma/client';

export class UsersRepository {
  constructor(private readonly db: PrismaClient) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.db.user.findFirst({
      where: { email, isActive: true },
    });
  }

  async create(data: Prisma.UserCreateInput): Promise<User> {
    return this.db.user.create({ data });
  }
}
```

**Never pass user-supplied values directly to Prisma `where` clauses with operators**:

```typescript
// ❌ Dangerous — user can inject Prisma query operators
await db.user.findFirst({ where: req.body }); // NoSQL injection

// ✅ Always use typed, specific fields
await db.user.findFirst({ where: { email: req.body.email } });
```

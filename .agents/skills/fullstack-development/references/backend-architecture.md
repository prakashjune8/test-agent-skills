# Backend Architecture & Security Guidelines

This guide details standards for building scalable, maintainable, and secure backend RESTful APIs using TypeScript with Node.js, Express, or Fastify.

---

## 1. Controller-Service-Repository Pattern

Maintain a strict three-tier architecture:

```
HTTP Request
     │
     ▼
[ Controller ]       <- Extracts params, validates DTO with Zod, formats HTTP response
     │
     ▼
[ Service ]          <- Contains business logic, calculations, orchestration, transactions
     │
     ▼
[ Repository / ORM ] <- Performs database queries (Prisma, Drizzle, TypeORM, raw SQL)
     │
     ▼
Database / External API
```

### Layer Rules:
1. **Controllers** never execute database queries directly.
2. **Services** never accept `req` or `res` objects from Express/Fastify. They accept plain JavaScript arguments/DTOs and return domain data or throw `AppError`.
3. **Repositories** isolate query syntax from business rules.

---

## 2. Request Validation with Zod

Create a reusable validation middleware:

```typescript
// server/src/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validateRequest = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    try {
      req.body = await schema.parseAsync(req.body);
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          success: false,
          error: {
            message: 'Input validation failed',
            statusCode: 400,
            details: error.flatten().fieldErrors,
          },
        });
      }
      next(error);
    }
  };
};
```

---

## 3. Centralized Error Handling Hierarchy

Define domain-specific errors extending a common `AppError`:

```typescript
// server/src/errors/AppError.ts
export class AppError extends Error {
  public readonly isOperational = true;

  constructor(
    public readonly message: string,
    public readonly statusCode: number = 500,
    public readonly details?: unknown
  ) {
    super(message);
    Object.setPrototypeOf(this, new.target.prototype);
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404);
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super(message, 403);
  }
}

export class BadRequestError extends AppError {
  constructor(message = 'Bad request', details?: unknown) {
    super(message, 400, details);
  }
}
```

---

## 4. Production Security Hardening

Always incorporate standard security middleware:

```typescript
// server/src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import { errorHandler } from './middleware/errorHandler';

const app = express();

// 1. Security Headers
app.use(helmet());

// 2. Controlled CORS
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS ? process.env.ALLOWED_ORIGINS.split(',') : 'http://localhost:5173',
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
}));

// 3. Rate Limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
  standardHeaders: true,
  legacyHeaders: false,
  message: { success: false, error: { message: 'Too many requests, please try again later.' } },
});
app.use('/api', limiter);

// 4. Body Parsing
app.use(express.json({ limit: '10kb' })); // Mitigate body overflow DoS
app.use(express.urlencoded({ extended: true, limit: '10kb' }));

// 5. Mount Routes & Error Middleware
// app.use('/api/v1', router);
app.use(errorHandler);

export default app;
```

---

## 5. Graceful Server Shutdown

Never terminate backend processes abruptly. Handle connection draining:

```typescript
// server/src/server.ts
import app from './app';
import { prisma } from './config/db';

const PORT = process.env.PORT || 4000;
const server = app.listen(PORT, () => {
  console.log(`Server listening on port ${PORT}`);
});

async function shutdown(signal: string) {
  console.log(`Received ${signal}. Gracefully closing HTTP server and DB connections...`);
  server.close(async () => {
    try {
      await prisma.$disconnect();
      console.log('Database connections closed. Exiting process.');
      process.exit(0);
    } catch (err) {
      console.error('Error during shutdown:', err);
      process.exit(1);
    }
  });

  // Force exit if not closed within 10s
  setTimeout(() => {
    console.error('Could not close connections in time, forcefully shutting down');
    process.exit(1);
  }, 10000);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

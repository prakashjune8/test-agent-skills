---
name: fullstack-development
description: >-
  Comprehensive guide and workflow for modern full-stack web application development
  using TypeScript and JavaScript. Covers frontend UI (React, Vite, Next.js, component
  architecture, modern CSS, accessibility, state management) and backend services (Node.js,
  Express, Fastify, REST/GraphQL APIs, database modeling, authentication, security, and
  end-to-end type safety). Use this skill whenever scaffolding, developing, refactoring,
  or reviewing full-stack web applications.
---

# Full-Stack Development Skill (UI & Backend)

This skill provides an end-to-end architectural guide, workflow runbook, and coding standards for developing production-grade web applications with modern TypeScript/JavaScript on both the frontend (UI) and backend.

---

## 1. Core Architectural Principles

1. **End-to-End Type Safety**:
   - Single source of truth for domain models and data transfer objects (DTOs).
   - Use schema validation libraries (such as **Zod**) to parse and validate requests at runtime and infer TypeScript types at compile time.
2. **Layered Separation of Concerns**:
   - **Frontend**: Strict separation of presentational primitives, business/container components, and data-fetching hooks.
   - **Backend**: Strict separation between Routing/Controllers (HTTP transport), Service Layer (business logic), and Data Access / Repository Layer (database / ORM).
3. **API Contract First**:
   - Establish request and response contracts before implementing UI or backend details.
   - Uniform response envelope: `{ success: true, data: T }` or `{ success: false, error: { message, code, details } }`.
4. **Resilience and Error Handling**:
   - Frontend: Error boundaries, skeleton loading states, graceful fallbacks, and user-friendly notifications.
   - Backend: Centralized error middleware, operational vs programmer error categorization, structured logging, and fail-safe defaults.
5. **Security by Default**:
   - Security headers (Helmet), explicit CORS policies, input validation/sanitization, rate limiting, and secure authentication (HTTP-only cookies or bearer tokens).

---

## 2. Recommended Directory Structure

For full-stack repositories (monorepo or split structure):

```text
├── packages/ or apps/
│   ├── client/                  # Frontend UI (React / Next.js / Vite)
│   │   ├── src/
│   │   │   ├── assets/          # Static assets (images, icons, fonts)
│   │   │   ├── components/      # UI components (atoms, molecules, organisms)
│   │   │   │   ├── ui/          # Core design system primitives (Button, Input, Modal)
│   │   │   │   └── features/    # Domain-specific composite components
│   │   │   ├── hooks/           # Custom React hooks (useAuth, useDebounce, etc.)
│   │   │   ├── services/        # API client and service integrations
│   │   │   ├── store/           # Global state management (Zustand, Context, etc.)
│   │   │   ├── types/           # Client-specific types
│   │   │   ├── utils/           # Helper functions & formatting utilities
│   │   │   ├── App.tsx          # Main application component & routes
│   │   │   ├── main.tsx         # Entry point
│   │   │   └── index.css        # Global CSS & design tokens
│   │   └── package.json
│   │
│   ├── server/                  # Backend API (Node.js / Express / Fastify)
│   │   ├── src/
│   │   │   ├── config/          # Environment variables & runtime configurations
│   │   │   ├── controllers/     # HTTP route handlers (extract params, call services)
│   │   │   ├── middleware/      # Auth, validation, error handler, rate limiter
│   │   │   ├── models/          # Database models / schemas (Prisma, Drizzle, etc.)
│   │   │   ├── routes/          # Express/Fastify route definitions
│   │   │   ├── services/        # Business logic & external API interactions
│   │   │   ├── utils/           # Backend helpers (jwt, logger, crypto)
│   │   │   ├── app.ts           # Express/Fastify app configuration
│   │   │   └── server.ts        # Server entry point & graceful shutdown
│   │   └── package.json
│   │
│   └── shared/                  # Shared schemas, DTOs & TypeScript contracts
│       ├── src/
│       │   ├── schemas/         # Zod schemas (user.schema.ts, auth.schema.ts)
│       │   └── types/           # Inferred types (export type User = z.infer<...>)
│       └── package.json
```

---

## 3. Step-by-Step Development Workflow

When implementing any full-stack feature, execute the following phases in order:

### Phase 1: Define Shared Contracts & Schemas
1. Define the Zod schema representing request payloads and response entities in `shared/` or `common/`.
2. Export the inferred TypeScript types.
```typescript
import { z } from 'zod';

export const CreateUserSchema = z.object({
  email: z.string().email('Invalid email address'),
  name: z.string().min(2, 'Name must be at least 2 characters'),
  role: z.enum(['admin', 'user']).default('user'),
});

export type CreateUserInput = z.infer<typeof CreateUserSchema>;
export type UserResponse = CreateUserInput & { id: string; createdAt: string };
```

### Phase 2: Build the Backend Service & Endpoints
1. **Validation Middleware**: Apply validation middleware with the Zod schema to reject malformed inputs early with HTTP 400.
2. **Service Layer**: Write pure business logic in a service class or module. Isolate database queries from HTTP concepts (`req`, `res`).
3. **Controller**: Handle HTTP input extraction, call the service, and return standardized responses using `sendSuccess()`.
4. **Error Handling**: Use custom errors (`NotFoundError`, `BadRequestError`, `UnauthorizedError`) and let the centralized error middleware catch and format the response.

### Phase 3: Build the Frontend UI Components
1. **Design System Tokens**: Ensure CSS custom properties or design system variables exist for colors, typography, spacing, and shadows.
2. **Atomic / UI Primitives**: Build or reuse base UI components (buttons, input fields, badges, dialogs) with keyboard navigation and ARIA attributes.
3. **Feature View / Page**: Assemble composite components with clean visual hierarchy, generous whitespace, modern typography, and responsive grid/flexbox layouts.
4. **State & Micro-Interactions**:
   - Provide visual feedback for hover, active, focus-visible, and disabled states.
   - Include loading skeletons, spinner indicators, and disabled buttons during pending submissions.

### Phase 4: Integration & Data Fetching
1. **API Client**: Use a centralized fetch wrapper or Axios instance configured with base URL, timeout, credentials, and interceptors.
2. **Server State**: Use **TanStack Query (React Query)** or standard hooks for querying and mutating data.
   - Cache invalidation on successful mutations.
   - Optimistic updates where appropriate for snappy user experience.
3. **Form Handling**: Integrate forms with React Hook Form + Zod resolver to share the exact same validation rules across client and server.

### Phase 5: Verification & Hardening
1. **Backend Verification**: Test endpoints with unit/integration tests or curl/fetch to verify edge cases (invalid inputs, missing tokens, duplicate keys).
2. **Frontend Verification**: Verify responsiveness across desktop, tablet, and mobile viewport widths.
3. **Security Check**: Confirm CORS, rate limiting, and authentication guards are active.

---

## 4. Quick Code Patterns

### Standard API Response Helper (Backend)
```typescript
// server/src/utils/response.ts
import { Response } from 'express';

export function sendSuccess<T>(res: Response, data: T, statusCode = 200, meta?: unknown) {
  return res.status(statusCode).json({
    success: true,
    data,
    ...(meta ? { meta } : {}),
  });
}

export function sendError(res: Response, message: string, statusCode = 500, details?: unknown) {
  return res.status(statusCode).json({
    success: false,
    error: {
      message,
      statusCode,
      details: details ?? null,
    },
  });
}
```

### Centralized Express Error Middleware
```typescript
// server/src/middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';

export class AppError extends Error {
  constructor(public message: string, public statusCode = 500, public details?: unknown) {
    super(message);
    this.name = 'AppError';
  }
}

export function errorHandler(err: unknown, req: Request, res: Response, next: NextFunction) {
  if (err instanceof ZodError) {
    return res.status(400).json({
      success: false,
      error: {
        message: 'Validation failed',
        statusCode: 400,
        details: err.flatten().fieldErrors,
      },
    });
  }

  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        message: err.message,
        statusCode: err.statusCode,
        details: err.details ?? null,
      },
    });
  }

  console.error('Unhandled Server Error:', err);
  return res.status(500).json({
    success: false,
    error: {
      message: 'Internal server error',
      statusCode: 500,
    },
  });
}
```

### Type-Safe API Client (Frontend)
```typescript
// client/src/services/apiClient.ts
interface ApiResponse<T> {
  success: boolean;
  data: T;
  error?: {
    message: string;
    statusCode: number;
    details?: unknown;
  };
}

export async function apiRequest<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, {
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
    ...options,
  });

  const body: ApiResponse<T> = await res.json();

  if (!res.ok || !body.success) {
    throw new Error(body.error?.message || `HTTP ${res.status}: Request failed`);
  }

  return body.data;
}
```

---

## 5. References and Deep Dives

For in-depth architectural patterns, consult:
- [UI Architecture & Styling Guidelines](./references/ui-architecture.md)
- [Backend Architecture & Security Guidelines](./references/backend-architecture.md)
- [API Contracts & End-to-End Type Safety](./references/api-contracts-and-types.md)

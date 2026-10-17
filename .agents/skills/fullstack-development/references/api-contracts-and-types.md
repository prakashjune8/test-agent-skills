# API Contracts & End-to-End Type Safety

This guide covers patterns for sharing schemas and types seamlessly between frontend and backend in TypeScript applications.

---

## 1. Unified Response Envelope

All API endpoints must return a predictable envelope:

```typescript
// Success Envelope
export interface ApiResponseSuccess<T> {
  success: true;
  data: T;
  meta?: {
    total?: number;
    page?: number;
    pageSize?: number;
    totalPages?: number;
    [key: string]: unknown;
  };
}

// Error Envelope
export interface ApiResponseError {
  success: false;
  error: {
    message: string;
    statusCode: number;
    details?: unknown;
  };
}

export type ApiResponse<T> = ApiResponseSuccess<T> | ApiResponseError;
```

---

## 2. Shared Zod Schemas & DTOs

Define schemas once in a `shared/` package or directory and import into both client and server:

```typescript
// shared/src/schemas/product.ts
import { z } from 'zod';

export const ProductSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1, 'Title is required').max(100),
  description: z.string().optional(),
  price: z.number().positive('Price must be greater than zero'),
  category: z.enum(['electronics', 'apparel', 'home', 'other']),
  isAvailable: z.boolean().default(true),
  createdAt: z.string().datetime(),
});

export const CreateProductDto = ProductSchema.omit({
  id: true,
  createdAt: true,
});

export const UpdateProductDto = CreateProductDto.partial();

export const ProductQueryDto = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
  category: z.enum(['electronics', 'apparel', 'home', 'other']).optional(),
  search: z.string().optional(),
});

// Inferred Types
export type Product = z.infer<typeof ProductSchema>;
export type CreateProductInput = z.infer<typeof CreateProductDto>;
export type UpdateProductInput = z.infer<typeof UpdateProductDto>;
export type ProductQueryParams = z.infer<typeof ProductQueryDto>;
```

---

## 3. Frontend Data Fetching with TanStack Query

Wrap API client calls in TanStack Query hooks for declarative caching, loading states, and mutations:

```tsx
// client/src/hooks/useProducts.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { apiRequest } from '../services/apiClient';
import type { Product, CreateProductInput } from '@app/shared';

export function useProducts() {
  return useQuery({
    queryKey: ['products'],
    queryFn: () => apiRequest<Product[]>('/api/v1/products'),
    staleTime: 1000 * 60 * 5, // 5 minutes fresh
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newProduct: CreateProductInput) =>
      apiRequest<Product>('/api/v1/products', {
        method: 'POST',
        body: JSON.stringify(newProduct),
      }),
    onSuccess: () => {
      // Invalidate and refetch products query
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

---

## 4. End-to-End Form Validation

Using React Hook Form with the Zod resolver ensures identical validation rules on client and server:

```tsx
// client/src/components/ProductForm.tsx
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreateProductDto, CreateProductInput } from '@app/shared';
import { Button } from './ui/Button';

interface ProductFormProps {
  onSubmit: (data: CreateProductInput) => void;
  isLoading?: boolean;
}

export const ProductForm: React.FC<ProductFormProps> = ({ onSubmit, isLoading }) => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<CreateProductInput>({
    resolver: zodResolver(CreateProductDto),
    defaultValues: {
      category: 'electronics',
      isAvailable: true,
    },
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="title" className="block text-sm font-medium">Title</label>
        <input id="title" {...register('title')} className="input-field" />
        {errors.title && <p className="text-danger text-xs mt-1">{errors.title.message}</p>}
      </div>

      <div>
        <label htmlFor="price" className="block text-sm font-medium">Price</label>
        <input id="price" type="number" step="0.01" {...register('price', { valueAsNumber: true })} className="input-field" />
        {errors.price && <p className="text-danger text-xs mt-1">{errors.price.message}</p>}
      </div>

      <Button type="submit" isLoading={isLoading}>
        Save Product
      </Button>
    </form>
  );
};
```

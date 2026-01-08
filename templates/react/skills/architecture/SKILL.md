# React Architecture - Principles and Organization

## Fundamental Architectural Principles

A well-designed architecture is the foundation of a maintainable and scalable React application. Key principles:

1. **Atomic Design**: Clear component hierarchy
2. **Separation of concerns**: Container/presenter, hooks, services
3. **Modularity**: Reusable components and hooks

Each file, folder, and component should have a single, well-defined responsibility:

- **Components**: Display and user interaction
- **Hooks**: Business logic and state management
- **Services**: API communication
- **Utils**: Pure utility functions
- **Types**: TypeScript definitions

Code should be organized into independent and reusable modules.

## Atomic Design Pattern

### Component Hierarchy

#### 1. Atoms

**Most basic components, not decomposable.**

```typescript
// components/atoms/Button/Button.tsx
import { FC, ButtonHTMLAttributes } from "react";
import { cva, type VariantProps } from "class-variance-authority";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors",
  {
    variants: {
      variant: {
        primary: "bg-blue-600 text-white hover:bg-blue-700",
        secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300",
        outline: "border border-gray-300 bg-transparent hover:bg-gray-100",
        ghost: "hover:bg-gray-100",
        danger: "bg-red-600 text-white hover:bg-red-700",
      },
      size: {
        sm: "h-9 px-3 text-sm",
        md: "h-10 px-4 text-base",
        lg: "h-11 px-8 text-lg",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "md",
    },
  }
);

export interface ButtonProps
  extends ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  isLoading?: boolean;
}

export const Button: FC<ButtonProps> = ({
  variant,
  size,
  isLoading,
  disabled,
  children,
  className,
  ...props
}) => {
  return (
    <button
      className={buttonVariants({ variant, size, className })}
      disabled={disabled || isLoading}
      {...props}
    >
      {isLoading ? <Spinner size="sm" /> : children}
    </button>
  );
};
```

```typescript
// components/atoms/Input/Input.tsx
import { FC, InputHTMLAttributes, forwardRef } from "react";
import { cn } from "@/utils/classnames";

export interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  error?: boolean;
  fullWidth?: boolean;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ error, fullWidth, className, ...props }, ref) => {
    return (
      <input
        ref={ref}
        className={cn(
          "px-3 py-2 border rounded-md outline-none transition-colors",
          "focus:ring-2 focus:ring-blue-500",
          error && "border-red-500 focus:ring-red-500",
          fullWidth && "w-full",
          className
        )}
        {...props}
      />
    );
  }
);

Input.displayName = "Input";
```

#### 2. Molecules

**Combination of several atoms.**

```typescript
// components/molecules/FormField/FormField.tsx
import { FC, ReactNode } from "react";
import { Input, InputProps } from "@/components/atoms/Input";
import { Label } from "@/components/atoms/Label";

export interface FormFieldProps extends InputProps {
  label: string;
  error?: string;
  helperText?: string;
  required?: boolean;
}

export const FormField: FC<FormFieldProps> = ({
  label,
  error,
  helperText,
  required,
  id,
  ...inputProps
}) => {
  const inputId = id || `field-${label.toLowerCase().replace(/\s/g, "-")}`;

  return (
    <div className="space-y-1">
      <Label htmlFor={inputId} required={required}>
        {label}
      </Label>
      <Input
        id={inputId}
        error={!!error}
        aria-invalid={!!error}
        aria-describedby={error ? `${inputId}-error` : undefined}
        {...inputProps}
      />
      {error && (
        <p id={`${inputId}-error`} className="text-sm text-red-600">
          {error}
        </p>
      )}
      {helperText && !error && (
        <p className="text-sm text-gray-500">{helperText}</p>
      )}
    </div>
  );
};
```

## Container/Presentational Pattern

### Logic/Presentation Separation

#### Container (Smart Component)

**Handles logic, side effects, state.**

```typescript
// features/products/components/ProductList/ProductListContainer.tsx
import { FC } from "react";
import { useProducts } from "@/features/products/hooks/useProducts";
import { ProductListPresenter } from "./ProductListPresenter";

export const ProductListContainer: FC = () => {
  const {
    products,
    isLoading,
    error,
    pagination,
    handlePageChange,
    handleSearch,
    handleSort,
  } = useProducts();

  if (error) {
    return <ErrorMessage error={error} />;
  }

  return (
    <ProductListPresenter
      products={products}
      isLoading={isLoading}
      pagination={pagination}
      onPageChange={handlePageChange}
      onSearch={handleSearch}
      onSort={handleSort}
    />
  );
};
```

#### Presenter (Dumb Component)

**Displays only, receives everything via props.**

```typescript
// features/products/components/ProductList/ProductListPresenter.tsx
import { FC } from "react";
import { Product } from "@/features/products/types";
import { DataTable } from "@/components/organisms/DataTable";
import { SearchBar } from "@/components/molecules/SearchBar";
import { Pagination } from "@/components/molecules/Pagination";

export interface ProductListPresenterProps {
  products: Product[];
  isLoading: boolean;
  pagination: PaginationState;
  onPageChange: (page: number) => void;
  onSearch: (query: string) => void;
  onSort: (field: string) => void;
}

export const ProductListPresenter: FC<ProductListPresenterProps> = ({
  products,
  isLoading,
  pagination,
  onPageChange,
  onSearch,
  onSort,
}) => {
  const columns = useMemo(
    () => [
      {
        accessorKey: "name",
        header: "Name",
      },
      {
        accessorKey: "price",
        header: "Price",
      },
      {
        accessorKey: "category",
        header: "Category",
      },
    ],
    []
  );

  return (
    <div className="space-y-4">
      <SearchBar onSearch={onSearch} placeholder="Search products..." />

      {isLoading ? (
        <Skeleton />
      ) : (
        <DataTable data={products} columns={columns} />
      )}

      <Pagination
        currentPage={pagination.currentPage}
        totalPages={pagination.totalPages}
        onPageChange={onPageChange}
      />
    </div>
  );
};
```

## Custom Hooks Organization

### Hook Structure

````typescript
// hooks/useExample/useExample.ts
import { useState, useEffect, useCallback } from "react";

export interface UseExampleOptions {
  initialValue?: string;
  onSuccess?: (data: string) => void;
}

export interface UseExampleReturn {
  value: string;
  isLoading: boolean;
  error: Error | null;
  update: (newValue: string) => void;
  reset: () => void;
}

/**
 * Custom hook to manage [description]
 *
 * @param options - Configuration options
 * @returns State and methods to manage [functionality]
 *
 * @example
 * ```tsx
 * const { value, update } = useExample({ initialValue: 'test' });
 * ```
 */
export const useExample = (
  options: UseExampleOptions = {}
): UseExampleReturn => {
  const { initialValue = "", onSuccess } = options;

  const [value, setValue] = useState(initialValue);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const update = useCallback(
    async (newValue: string) => {
      setIsLoading(true);
      setError(null);

      try {
        // Business logic
        setValue(newValue);
        onSuccess?.(newValue);
      } catch (err) {
        setError(err as Error);
      } finally {
        setIsLoading(false);
      }
    },
    [onSuccess]
  );

  const reset = useCallback(() => {
    setValue(initialValue);
    setError(null);
  }, [initialValue]);

  return {
    value,
    isLoading,
    error,
    update,
    reset,
  };
};
````

## Architecture Best Practices

### 1. Index Barrels

**Facilitate imports with index files.**

```typescript
// features/orders/components/index.ts
export { OrderList } from "./OrderList";
export { OrderDetail } from "./OrderDetail";
export { OrderForm } from "./OrderForm";

// features/orders/hooks/index.ts
export { useOrders, useOrder } from "./useOrders";
export {
  useCreateOrder,
  useUpdateOrder,
  useCancelOrder,
} from "./useOrderMutations";
export { useOrderFilters } from "./useOrderFilters";

// features/orders/index.ts
export * from "./components";
export * from "./hooks";
export * from "./types";
```

**Usage**:

```typescript
// Instead of multiple imports
import { OrderList } from "@/features/orders/components/OrderList";
import { OrderDetail } from "@/features/orders/components/OrderDetail";

// Single import
import { OrderList, OrderDetail } from "@/features/orders/components";
```

### 2. Absolute Imports

**tsconfig.json configuration**:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/features/*": ["./src/features/*"],
      "@/hooks/*": ["./src/hooks/*"],
      "@/utils/*": ["./src/utils/*"],
      "@/types/*": ["./src/types/*"]
    }
  }
}
```

**Usage**:

```typescript
// ❌ Bad - relative imports
import { Button } from "../../../components/atoms/Button";

// ✅ Good - absolute imports
import { Button } from "@/components/atoms/Button";
```

### 3. Lazy Loading

**Route code splitting**:

```typescript
// app/router.tsx
import { lazy, Suspense } from "react";
import { createBrowserRouter } from "react-router-dom";
import { Spinner } from "@/components/atoms/Spinner";

// Lazy load pages
const HomePage = lazy(() => import("@/pages/HomePage"));
const DashboardPage = lazy(() => import("@/pages/DashboardPage"));
const CatalogPage = lazy(() => import("@/features/products/pages/CatalogPage"));

export const router = createBrowserRouter([
  {
    path: "/",
    element: (
      <Suspense fallback={<Spinner />}>
        <HomePage />
      </Suspense>
    ),
  },
  {
    path: "/dashboard",
    element: (
      <Suspense fallback={<Spinner />}>
        <DashboardPage />
      </Suspense>
    ),
  },
  {
    path: "/catalog",
    element: (
      <Suspense fallback={<Spinner />}>
        <CatalogPage />
      </Suspense>
    ),
  },
]);
```

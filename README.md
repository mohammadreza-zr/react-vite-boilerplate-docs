# 📘 Developer Guide & Architecture Documentation

Welcome to the **Frontend Boilerplate** development guide. This document is the single source of truth for developers working on this project. It covers architecture, conventions, API patterns, and step-by-step guides for common tasks.

> **Target Audience:** Frontend Developers, Tech Leads, and Architects
> **Last Updated:** June 2026
> **Project Type:** Enterprise-grade React SPA Boilerplate

---

## 📑 Table of Contents

1. [Getting Started](#1-getting-started)
2. [Available Scripts](#2-available-scripts)
3. [Project Structure](#3-project-structure)
4. [Naming Conventions](#4-naming-conventions)
5. [Architecture Overview](#5-architecture-overview)
6. [API & Data Fetching (Critical)](#6-api--data-fetching-critical)
7. [UI System & Atomic Design](#7-ui-system--atomic-design)
8. [Forms & Validation](#8-forms--validation)
9. [Routing](#9-routing)
10. [Adding a New Feature (Step-by-Step)](#10-adding-a-new-feature-step-by-step)
11. [Storybook](#11-storybook)
12. [Git Workflow & Commit Convention](#12-git-workflow--commit-convention)
13. [Important Rules & Gotchas](#13-important-rules--gotchas)
14. [The Golden Rule of the UI System](#14-the-golden-rule-of-the-ui-system)

---

## 1. Getting Started

### Prerequisites

- Node.js 20+ (LTS recommended)
- npm 10+ or yarn 4+

### Installation

```bash
npm install
```

### Development Server

```bash
npm run dev
```

The app will be available at `http://localhost:5173`.

---

## 2. Available Scripts

| Command                   | Description                                          |
| ------------------------- | ---------------------------------------------------- |
| `npm run dev`             | Start the development server with HMR                |
| `npm run build`           | Build for production (TypeScript check + Vite build) |
| `npm run preview`         | Preview the production build locally                 |
| `npm run lint`            | Run ESLint on the entire codebase                    |
| `npm run storybook`       | Start Storybook at `http://localhost:6006`           |
| `npm run build-storybook` | Build Storybook for deployment                       |

---

## 3. Directory Structure

```ts
src/
├── core/           # Global infrastructure and reusables (NO business logic)
│   ├── constants/  # Global constants (time, translations, etc.)
│   ├── data/       # Static data (e.g., mime-types)
│   ├── hooks/      # Global custom hooks (use-mobile, use-direction)
│   ├── lib/        # Utility libraries (Zod config, compression, utils)
│   ├── providers/  # Global providers (e.g., QueryClientProvider)
│   ├── schemas/    # Global validation schemas (Zod)
│   ├── services/   # Service classes (e.g., api-service.ts)
│   ├── store/      # Global Zustand stores (e.g., user.store.ts)
│   ├── types/      # Global interfaces and types
│   └── ui/         # 🎨 The ACTUAL Design System (Atomic Design)
├── features/       # Business modules (Auth, Users, Dashboard, etc.)
│   └── [feature]/
│       ├── api/        # API functions and React Query keys
│       ├── components/ # Feature-specific components
│       ├── hooks/      # Feature-specific hooks
│       └── types/      # Feature-specific types
├── pages/          # Route-level composition only (Filename matches route)
├── router/         # React Router configuration (public/private routes)
├── assets/         # Static assets (fonts, images)
├── scripts/        # Automation scripts (e.g., create-image-index.js)
└── stories/        # Storybook configuration and component stories
```

### Key Directories Explained

#### `core/`

The `core` directory contains all shared application resources that can be reused across multiple features.

Core is not limited to framework infrastructure. Any reusable logic, service, schema, type, hook, UI component, or utility that is shared between multiple modules should live in `core`.

Examples:

- Shared UI components
- Shared hooks
- Shared schemas
- Shared types
- Shared services
- Shared state stores
- Global providers
- Utility functions

A resource should only be moved to `core` when it is genuinely shared across unrelated modules.

If a resource is only needed by a single feature, it must remain inside that feature.

#### `features/`

Each feature is a **self-contained module** with its own:

```ts
features/user-management/
├── api/
├── components/
├── hooks/
├── schemas/
├── services/
├── store/
├── constants/
├── lib/
├── types/
└──  sub-features/  #only when feature is very large
    ├── user-list/
    │   ├── api/
    │   ├── components/
    │   └── types/
    │
    ├── user-permissions/
    │   ├── api/
    │   ├── components/
    │   └── types/
    │
    └── user-roles/
        ├── api/
        ├── components/
        └── types/
```

#### `ui/`

Follows **Atomic Design** methodology:

- `atoms/` → Basic components (Button, Input, Checkbox, etc.)
- `molecules/` → Combinations of atoms (FormField, Combobox, Pagination)
- `organisms/` → Complex sections (FormBuilder, Sidebar)
- `templates/` → Page layouts

#### `pages/`

**Composition-only layer.** Pages should only import and render features. No business logic.

```tsx
// ✅ CORRECT
export default function UsersPage() {
  return <UserManagementFeature />;
}

// ❌ WRONG
export default function UsersPage() {
  const [users, setUsers] = useState([]); // Business logic belongs in features/
  // ...
}
```

---

## 4. Naming Conventions

These rules are **enforced by ESLint** and will cause build errors if violated.

| Type                  | Convention         | Example                                |
| --------------------- | ------------------ | -------------------------------------- |
| Files & Folders       | `kebab-case`       | `user-profile.tsx`, `user-management/` |
| Components            | `PascalCase`       | `UserProfile`, `UserManagementFeature` |
| Hooks                 | `useSomething.ts`  | `use-mobile.ts`, `use-auth.ts`         |
| Features              | `lowercase`        | `authentication/`, `billing/`          |
| Variables & Functions | `camelCase`        | `getUserById`, `isLoading`             |
| Constants             | `UPPER_SNAKE_CASE` | `MAX_RETRY_COUNT`                      |
| Types & Interfaces    | `PascalCase`       | `UserResponse`, `LoginFormValues`      |

---

## 5. Architecture Overview

### 5.1 Layer Separation

The project enforces strict layer separation via `eslint-plugin-boundaries`:

```ts
pages/ → can import → features/, ui/
features/ → can import → core/, ui/
ui/ → can import → core/ (only utils/types)
core/ → cannot import → features/, pages/, ui/
```

### 5.2 State Management

- **Server State:** TanStack React Query (with persistence support)
- **UI State:** Local component state (`useState`, `useReducer`)
- **Global UI State:** React Context (only when absolutely necessary)

> ⚠️ **No Redux, or other global state libraries** unless explicitly approved.(only zustand allowed)

---

## 6. API & Data Fetching (Critical)

This is the **most important section**. Read carefully.

### 6.1 The `apiService` Singleton

Located at `core/services/api-service.ts`, this is a **custom Axios-based class** that handles:

1. **Automatic Token Refresh:** When a 401 occurs, it pauses the request, refreshes the token, and retries.
2. **Request Queuing:** If multiple requests fail with 401 simultaneously, they are queued. Once the token is refreshed, all queued requests are retried with the new token.
3. **Dynamic Path Parameters:** Automatically replaces `{id}` in URLs with actual values.
4. **Type Safety:** Full TypeScript support for request body and response.

**Usage:**

```typescript
import { apiService } from "@/core/services";

// GET request with typed response
const response = await apiService.get<UserResponse>("/users/123");

// POST request with typed body and response
const newUser = await apiService.post<UserResponse, CreateUserDto>("/users", {
  name: "John",
  email: "john@example.com",
});

// Dynamic path parameters
const user = await apiService.get<UserResponse>(
  "/users/{id}",
  {
    addTemplateToUrl: {
        id: 123 // Automatically replaces {id} with 123
    }
  },
);

const user = await apiService.get<UserResponse>(
  "/users",
  {
    addToUrl: [123] // Automatically add to url: /users/123
  },
);
```

### 6.2 The `*keys` Pattern for React Query

Every feature **must** define a `keys` object to manage React Query keys. This ensures consistency between query keys and API endpoints.

**Example: `features/users/api/user.keys.ts`**

```typescript
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  list: (filters: string) => [...userKeys.lists(), { filters }] as const,
  details: () => [...userKeys.all, "detail"] as const,
  detail: (id: number) => [...userKeys.details(), id] as const,
};
```

### 6.3 Complete Example: Fetching Data in a Feature

## **Step 1: Define types**

```typescript
// features/users/types/user.types.ts
export interface User {
  id: number;
  name: string;
  email: string;
}

export interface CreateUserDto {
  name: string;
  email: string;
}
```

## **Step 2: Define query keys**

```typescript
// features/users/api/user.keys.ts
export const userKeys = {
  all: ["users"] as const,
  lists: () => [...userKeys.all, "list"] as const,
  list: (filters: string) => [...userKeys.lists(), { filters }] as const,
  details: () => [...userKeys.all, "detail"] as const,
  detail: (id: number) => [...userKeys.details(), id] as const,
};
```

## **Step 3: Create API hooks**

```typescript
// features/users/api/user.api.ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";

import { apiService } from "@/core/services";

import type { CreateUserDto, User } from "../types/user.types";
import { userKeys } from "./user.keys";

// GET single user
export const useGetUser = (id: number) => {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => apiService.get<User>("/users/{id}", { id }),
    enabled: !!id,
  });
};

// GET list of users
export const useGetUsers = (filters: string) => {
  return useQuery({
    queryKey: userKeys.list(filters),
    queryFn: () => apiService.get<User[]>("/users", { filters }),
  });
};

// CREATE user
export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateUserDto) =>
      apiService.post<User, CreateUserDto>("/users", data),
    onSuccess: () => {
      // Invalidate all user lists to refetch
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
    },
  });
};
```

## **Step 4: Use in component**

```typescript
// features/users/components/UserList.tsx
import { useGetUsers } from '../api/user.api';

export function UserList() {
  const { data: users, isLoading } = useGetUsers('active');

  if (isLoading) return <Spinner />;

  return (
    <ul>
      {users?.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

---

## 7. UI System & Atomic Design

### 7.1 Component Hierarchy

```ts
Atoms → Molecules → Organisms → Templates
```

- **Atoms:** `Button`, `Input`, `Checkbox`, `Label`, `Spinner`
- **Molecules:** `FormField`, `Combobox`, `Pagination`, `Dialog`
- **Organisms:** `FormBuilder`, `Sidebar`, `DataTable`
- **Templates:** `PublicLayout`, `AuthenticatedLayout`

### 7.2 Using UI Components

All UI components are built with **shadcn/ui** and **Radix UI**. They are fully customizable and owned by the project (no vendor lock-in).

```tsx
import { Button } from "@/ui/atoms/button";
import { Input } from "@/ui/atoms/input";
import { FormField } from "@/ui/molecules/field";

function LoginForm() {
  return (
    <form>
      <FormField label="Email">
        <Input type="email" placeholder="Enter your email" />
      </FormField>
      <Button variant="primary">Sign In</Button>
    </form>
  );
}
```

### 7.3 Component Variants

Components use `class-variance-authority (cva)` for managing variants:

```tsx
// ui/atoms/button/variants.ts
import { cva } from "class-variance-authority";

export const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors",
  {
    variants: {
      variant: {
        primary: "bg-primary text-white hover:bg-primary/90",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        destructive: "bg-destructive text-white hover:bg-destructive/90",
        outline: "border border-input bg-background hover:bg-accent",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "default",
    },
  },
);
```

---

## 8. Forms & Validation

### 8.1 Standard Stack

- **React Hook Form** for form state management
- **Zod** for schema validation
- **@hookform/resolvers** for integration

### 8.2 Example: Creating a Form

```tsx
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

import { useForm } from "react-hook-form";

import { Button } from "@/ui/atoms/button";
import { Input } from "@/ui/atoms/input";
import { FormField } from "@/ui/molecules/field";

// 1. Define schema
const loginSchema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

type LoginFormValues = z.infer<typeof loginSchema>;

// 2. Use in component
export function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<LoginFormValues>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = (data: LoginFormValues) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <FormField label="Email" error={errors.email?.message}>
        <Input type="email" {...register("email")} />
      </FormField>

      <FormField label="Password" error={errors.password?.message}>
        <Input type="password" {...register("password")} />
      </FormField>

      <Button type="submit">Sign In</Button>
    </form>
  );
}
```

### 8.3 FormBuilder (Advanced)

For complex, dynamic forms, use the `FormBuilder` organism located at `ui/organisms/form-builder/`. It supports:

- Dynamic field rendering
- Conditional logic (dependencies)
- Grid layouts
- Custom field types

---

## 9. Routing

### 9.1 Route Structure

Routes are defined in `src/router/`:

- `public-routes.tsx` → Routes accessible without authentication
- `private-routes.tsx` → Routes requiring authentication
- `router.tsx` → Main router configuration

### 9.2 Adding a New Route

## **Step 1: Create the page**

```tsx
// src/pages/settings.tsx
import { SettingsFeature } from "@/features/settings";

export default function SettingsPage() {
  return <SettingsFeature />;
}
```

## **Step 2: Register the route**

```tsx
// src/router/private-routes.tsx
import { lazy } from "react";

const SettingsPage = lazy(() => import("@/pages/settings"));

export const privateRoutes = [
  {
    path: "/settings",
    element: <SettingsPage />,
  },
  // ... other routes
];
```

### 9.3 Lazy Loading

All pages should be **lazy-loaded** using `React.lazy()` for optimal performance.

---

## 10. Adding a New Feature (Step-by-Step)

Let's add a new feature called `billing`.

### Step 1: Create Feature Folder

```bash
mkdir -p src/features/billing/{components,hooks,api,types}
```

### Step 2: Define Types

```typescript
// src/features/billing/types/billing.types.ts
export interface Invoice {
  id: number;
  amount: number;
  status: "paid" | "pending";
  createdAt: string;
}
```

### Step 3: Define Query Keys

```typescript
// src/features/billing/api/billing.keys.ts
export const billingKeys = {
  all: ["billing"] as const,
  invoices: () => [...billingKeys.all, "invoices"] as const,
  invoice: (id: number) => [...billingKeys.invoices(), id] as const,
};
```

### Step 4: Create API Hooks

```typescript
// src/features/billing/api/billing.api.ts
import { useQuery } from "@tanstack/react-query";

import { apiService } from "@/core/services";

import type { Invoice } from "../types/billing.types";
import { billingKeys } from "./billing.keys";

export const useGetInvoices = () => {
  return useQuery({
    queryKey: billingKeys.invoices(),
    queryFn: () => apiService.get<Invoice[]>("/billing/invoices"),
  });
};
```

### Step 5: Create Components

```tsx
// src/features/billing/components/InvoiceList.tsx
import { useGetInvoices } from "../api/billing.api";

export function InvoiceList() {
  const { data: invoices, isLoading } = useGetInvoices();

  if (isLoading) return <div>Loading...</div>;

  return (
    <ul>
      {invoices?.map((invoice) => (
        <li key={invoice.id}>
          ${invoice.amount} - {invoice.status}
        </li>
      ))}
    </ul>
  );
}
```

### Step 6: Create Feature Entry Point

```tsx
// src/features/billing/index.tsx
import { InvoiceList } from "./components/InvoiceList";

export function BillingFeature() {
  return (
    <div>
      <h1>Billing</h1>
      <InvoiceList />
    </div>
  );
}
```

### Step 7: Create Page

```tsx
// src/pages/billing.tsx
import { BillingFeature } from "@/features/billing";

export default function BillingPage() {
  return <BillingFeature />;
}
```

### Step 8: Register Route

```tsx
// src/router/private-routes.tsx
const BillingPage = lazy(() => import("@/pages/billing"));

export const privateRoutes = [
  {
    path: "/billing",
    element: <BillingPage />,
  },
];
```

✅ **Done!** Your new feature is now integrated.

---

## 11. Storybook

Storybook is used for **developing and documenting UI components in isolation**.

### Running Storybook

```bash
npm run storybook
```

### Writing a Story

```typescript
// ui/atoms/button/Button.stories.ts
import type { Meta, StoryObj } from "@storybook/react";

import { Button } from "./button";

const meta = {
  title: "Atoms/Button",
  component: Button,
  tags: ["autodocs"],
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: {
    variant: "primary",
    children: "Click me",
  },
};

export const Secondary: Story = {
  args: {
    variant: "secondary",
    children: "Click me",
  },
};
```

---

## 12. Git Workflow & Commit Convention

### 12.1 Branch Naming

```ts
feature / feature - name;
bugfix / issue - description;
hotfix / critical - fix;
refactor / module - name;
```

### 12.2 Commit Messages

We enforce **Conventional Commits** via `commitlint` and `husky`.

**Format:**

```ts
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**

- `feat` → New feature
- `fix` → Bug fix
- `docs` → Documentation changes
- `style` → Code style changes (formatting, semicolons, etc.)
- `refactor` → Code refactoring (no feature or bug changes)
- `test` → Adding or updating tests
- `chore` → Maintenance tasks (dependencies, configs, etc.)

**Examples:**

```bash
feat(auth): add login form with validation
fix(api): resolve token refresh race condition
docs(readme): update installation instructions
refactor(ui): simplify button component variants
test(user): add unit tests for user API hooks
```

### 12.3 Pre-commit Hooks

**Husky** and **lint-staged** automatically run:

- ESLint on staged files
- Prettier formatting
- TypeScript type checking

If any check fails, the commit will be **rejected**.

---

## 13. Important Rules & Gotchas

### ✅ DO

- ✅ Always use `apiService` for API calls (never use `axios` directly)
- ✅ Define query keys in `*keys.ts` files
- ✅ Keep business logic in `features/`, not in `pages/`
- ✅ Use TypeScript strict mode (no `any` types)
- ✅ Write Storybook stories for all new UI components
- ✅ Follow Atomic Design principles when creating UI components
- ✅ Use `React.lazy()` for all page components

### ❌ DON'T

- ❌ Don't import from `features/` into `core/` or `ui/`
- ❌ Don't use global state (Redux, Zustand) unless absolutely necessary
- ❌ Don't bypass the `apiService` singleton
- ❌ Don't write business logic in `pages/`
- ❌ Don't use `any` type (use `unknown` or proper types)
- ❌ Don't skip TypeScript types for API responses
- ❌ Don't commit without running `npm run lint`

### ⚠️ Common Pitfalls

## 14. The Golden Rule of the UI System

UI component management in this project is strictly divided into two distinct areas:

```txt
1. Root ui/ folder (or outside core):
  - This folder is strictly the target for the shadcn/ui CLI installation commands.
  - ⛔ STRICT PROHIBITION: Direct imports from this folder anywhere in the application code are forbidden.

2. core/ui/ folder:
  - ✅ Single Source of Truth: This is the final, customized, and enterprise-grade design system.
  - Raw components from the root ui/ folder are moved here, refactored, and organized according to Atomic Design principles.
  - Atomic Design Structure:
     . atoms/: Base, indivisible components (Button, Input, Table, Card, Checkbox).
     . molecules/: Combinations of atoms (Combobox, Dialog, Field, Pagination).
     . organisms/: Complex, independent components (FormBuilder, Sidebar).
     . layouts/: Page-level structural templates (PublicLayout).
```

```json
Rule: All UI imports across the entire project must originate from @core/ui/....
```

1. **Token Refresh Issues:** If you see multiple 401 errors, ensure you're using `apiService` and not raw `axios`.
2. **Query Key Mismatches:** Always use the `*keys` object. Don't hardcode query keys.
3. **Circular Dependencies:** The `eslint-plugin-boundaries` will catch this. Respect the layer separation.
4. **Performance Issues:** Always lazy-load pages. Use `React.memo()` for expensive components.

---

## 📞 Need Help?

- **Architecture Questions:** Contact the Tech Lead
- **Bug Reports:** Create an issue on GitHub/GitLab
- **Feature Requests:** Discuss in the team meeting first

---

## 📚 Additional Resources

- [React Documentation](https://react.dev/)
- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [shadcn/ui Documentation](https://ui.shadcn.com/)
- [Radix UI Documentation](https://www.radix-ui.com/)
- [Zod Documentation](https://zod.dev/)

---

### **Happy Coding! 🚀**

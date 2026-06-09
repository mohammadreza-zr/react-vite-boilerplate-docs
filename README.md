# 📘 Developer Guide & Architecture Documentation

Welcome to the **Frontend Boilerplate** development guide. This document is the single source of truth for architecture, conventions, and development rules.

> Target Audience: Frontend Developers, Tech Leads, Architects
> Project Type: Enterprise-grade React + TypeScript SPA Boilerplate

---

# Table of Contents
1. Architecture Principles
2. Project Structure
3. Core Concept
4. Features & Sub-features
5. Pages Layer
6. Module Boundaries
7. Import Rules
8. UI System (Atomic Design)
9. State Management
10. API Layer
11. Forms & Validation
12. Routing
13. Adding New Features
14. ESLint Enforcement
15. Golden Rules

---

# 1. Architecture Principles

## 1.1 Feature Independence
Each feature is a self-contained business module.

Examples:
- auth
- users
- billing
- reports

❌ Features must NOT depend on other features.

If sharing is needed → move to Core.

---

## 1.2 Core is Shared Infrastructure
Core contains reusable, cross-feature logic only.

Includes:
- UI components
- hooks
- services
- utils
- schemas
- types
- store

❌ No business logic in Core.

---

## 1.3 Sub-features
Large domains can be split internally.

Example:
user-management/
  users/
  roles/
  permissions/

Sub-features behave like features but stay inside parent domain.

---

## 1.4 Pages = Composition Only
Pages only compose features.

```tsx
export default function UsersPage() {
  return <UserManagement />;
}
```

❌ No business logic in pages.

---

# 2. Project Structure

```
src/
├── core/
├── features/
├── pages/
├── router/
├── assets/
└── main.tsx
```

---

# 3. Core

Core is shared infrastructure used across features.

Allowed:
- api services
- design system
- hooks
- utils
- schemas
- types
- global store

Rule: If only one feature uses it → DO NOT put in core.

---

# 4. Features

Each feature owns:
- business logic
- API calls
- state
- UI specific to domain

Example:
features/users/
  api/
  components/
  hooks/
  store/
  types/

---

# 5. Sub-features

Used when a feature grows large.

Example:
features/user-management/
  users/
  roles/
  permissions/

Each sub-feature can have its own api/hooks/components.

---

# 6. Pages

Pages are route entry points.

Responsibilities:
- routing composition
- layout composition

❌ No logic

---

# 7. Module Boundaries

Allowed imports:
- feature → core
- feature → ui
- page → feature

Not allowed:
- feature → feature
- sub-feature → other feature

If needed → move shared logic to Core.

---

# 8. UI System (Atomic Design)

core/ui/
- atoms
- molecules
- organisms
- templates

All UI must come from Core UI system.

---

# 9. State Management

- Server state: React Query
- UI state: local state
- Global UI state: Zustand (only if necessary)

---

# 10. API Layer

Centralized API service handles:
- authentication
- token refresh
- request queueing
- retry logic

Behavior:
If token expires:
1. first request triggers refresh
2. others wait
3. all retry after refresh

---

# 11. Forms & Validation

Use Zod for validation.

Feature-owned schemas preferred.
Shared schemas go to Core.

---

# 12. Routing

Pages are lazy-loaded:

```ts
lazy(() => import("@/pages/users"));
```

---

# 13. Adding New Feature

Steps:
1. Create feature folder
2. Add api/hooks/store/types only if needed
3. Export default feature
4. Add page wrapper

Do NOT create empty folders.

---

# 14. ESLint Enforcement

Enforces:
- no cross-feature imports
- naming conventions
- module boundaries
- export rules

---

# 15. Golden Rules

1. Features are independent
2. Core is shared only
3. Pages are composition only
4. No cross-feature imports
5. Shared logic → Core
6. Feature owns its business logic

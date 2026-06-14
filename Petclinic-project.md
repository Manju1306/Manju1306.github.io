# Building Spring PetClinic in TypeScript: React 19, Express 5 & Modern Auth

I took the classic **Spring PetClinic** - originally created by **Pivotal** (now VMware Tanzu) and the most well-known demo app in the Java ecosystem - and built it entirely in **TypeScript**. No Java. No Spring. A modern full-stack TypeScript application with a beautiful UI, production-grade authentication, and an AI chat assistant.

Here's how I built it and what I learned along the way.

---

## Why Build PetClinic?

The original [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) by **Pivotal** has been the go-to sample app for Java developers since 2003. It demonstrates how to build a CRUD application with owners, pets, veterinarians, and visits. It's been forked thousands of times, used in countless tutorials, and remains the standard way to learn Spring.

I wanted to answer a question: **What does the Pivotal PetClinic look like when built with the modern TypeScript ecosystem?**

Not just a port. A complete rethink - keeping the same domain model and data schema that Pivotal designed, but reimagining the architecture with the best tools available in 2025:

- **React 19** with the latest router and hooks patterns
- **Express 5** (yes, it's finally here) with proper async error handling
- **Tailwind CSS v4** with a custom design system
- **JWT authentication** with refresh token rotation
- **AI-powered chat** with multi-provider LLM support

---

## The Architecture

```
petclinic-project/
├── backend/           # TypeScript + Express 5 REST API
│   ├── src/
│   │   ├── ai/        # AI chat service + MCP tools
│   │   ├── auth/      # JWT + refresh tokens + password reset
│   │   ├── models/    # Sequelize models (single file)
│   │   ├── routes/    # One router per resource
│   │   └── server.ts  # Entry point with graceful shutdown
│   └── petclinic.db   # SQLite database
│
├── frontend/          # React 19 SPA
│   └── client/
│       ├── src/
│       │   ├── auth/       # AuthContext, token refresh, guards
│       │   ├── components/ # Feature folders (owners, pets, vets, visits)
│       │   ├── styles/     # Tailwind v4 + custom theme
│       │   └── types/      # Shared interfaces
│       └── vite.config.ts
```

### Backend: Express 5 + SQLite

The backend is a REST API serving the classic PetClinic schema designed by Pivotal: **owners, pets, vets, specialties, and visits**. I kept the same database structure — it's a well-designed domain model - but rewrote the entire server layer. Key decisions:

**Express 5** - The long-awaited major version finally handles async errors properly. Combined with a custom `asyncHandler` wrapper and centralized error mapping (Sequelize errors to HTTP statuses), error handling is clean and consistent.

**Single-file models** - All Sequelize models live in one file (`petclinic.models.ts`). With TypeScript's `InferAttributes` and `InferCreationAttributes`, the models are fully typed without separate interface files. The classic PetClinic schema maps naturally:

- `Owner` has many `Pet`
- `Pet` belongs to `PetType`, has many `Visit`
- `Vet` belongs to many `Specialty` (through `VetSpecialty`)
- `User` has many `Role` (natural key on `username`)

**OpenAPI documentation** - Every endpoint is documented in a hand-written `openapi.yaml`, served via Swagger UI at `/api/docs`. No code generation — the spec is the source of truth.

### Frontend: React 19 + Tailwind v4

The frontend is a single-page app with a design language inspired by editorial print — warm tones, serif typography, and intentional whitespace.

**No state management library** - Each page fetches its own data with `useEffect` + local `useState`. For a CRUD app of this scale, global state would add complexity without solving a real problem.

**Feature folders** - Each resource (`owners/`, `pets/`, `vets/`, `visits/`) follows a consistent split:
- `XxxPage.tsx` - Route container, data fetching, URL params
- `XxxEditor.tsx` - Controlled form, submit logic, navigation
- Presentational siblings - Take fully-resolved props, no data fetching

**Shared form primitives** - `Input`, `SelectInput`, `DateInput` components handle validation, error display, and constraints (`NotEmpty`, `Digits(n)`) consistently across all editors.

**shadcn/ui components** - Built a custom component library (`Card`, `Button`, `Badge`, `Alert`, `Input`, `Label`) using `class-variance-authority` for type-safe variants, integrated with Tailwind v4's CSS variable system.

---

## Authentication: Done Right

This isn't a demo login screen bolted on top. It's a production-grade auth system:

### Stateless Access Tokens + Stateful Refresh Tokens

- **Access tokens** are short-lived JWTs (15 min) carrying `{ username, roles }`. Verified statelessly by middleware - no database hit on every request.
- **Refresh tokens** are opaque 32-byte hex strings, stored as SHA-256 hashes in the database. They support:
  - **Token rotation** - Each refresh revokes the old token and issues a new one
  - **Reuse detection** - Reusing a rotated token returns 401 (potential theft detected)
  - **Role propagation** - Roles are re-read from the database on refresh, so role changes take effect within one refresh cycle

### Password Reset Flow

- `POST /auth/password-reset/request` always returns 204 (no email enumeration)
- Single-use tokens with configurable TTL
- Password change revokes all refresh tokens across all sessions
- The initiating session gets a fresh token pair so the user stays signed in

### Frontend Auth Flow

- `AuthContext` provides `user`, `isAuthenticated`, `login()`, `logout()`, `signup()`
- `RequireAuth` and `RequireAdmin` route guards
- Automatic token refresh via `apiFetch` — when an access token expires, the refresh client transparently gets a new one before retrying

---

## The Design System

The UI uses a custom **Tailwind v4 theme** with CSS variables for consistent branding:

```css
@theme {
  --color-spring-green: #6db33f;
  --color-spring-dark-green: #5fa134;
  --color-spring-brown: #34302d;
  --color-paper: #f5f0eb;
  --color-paper-deep: #ebe5dc;
}
```

The welcome page features a hero section with animated paw silhouette, editorial-style quick action cards with hover shadows, and a receipt-style daily schedule.

Pet cards use emoji icons mapped by type (dog, cat, bird, snake, lizard, hamster), badge components for pet types, hover-to-reveal delete buttons on visits, and responsive grid layouts (1/2/3 columns).

---

## What I Learned

1. **Express 5 is ready.** Async error handling works. The migration from Express 4 is minimal. If you're starting a new project, use it.

2. **You don't need a state management library for every React app.** Local state + prop drilling works fine for CRUD apps. The code is simpler, more readable, and easier to debug.

3. **Tailwind v4 is a significant upgrade.** The Vite plugin, CSS variables in `@theme`, and the new `@layer` system make it feel like a native CSS extension rather than a utility framework bolted on.

4. **Auth is hard, but it's important to do it right.** Refresh token rotation with reuse detection is the baseline for any production app. Libraries help, but understanding the flow is non-negotiable.

5. **TypeScript end-to-end is a superpower.** Sharing mental models between frontend and backend - same language, same patterns, same tooling — eliminates an entire class of context-switching overhead.

---

## Tech Stack Summary

| Layer | Technology |
|-------|-----------|
| **Frontend** | React 19, TypeScript, Vite, Tailwind CSS v4, React Router v7 |
| **Backend** | Express 5, TypeScript, Sequelize 6, SQLite |
| **Auth** | JWT access tokens, stateful refresh tokens, bcrypt password hashing |
| **UI Components** | shadcn/ui pattern (CVA + Tailwind), custom theme |
| **AI Chat** | Vercel AI SDK v6, Google Gemini / OpenAI / Anthropic |
| **API Docs** | OpenAPI 3.0, Swagger UI |
| **Testing** | Vitest, Testing Library |
| **Package Manager** | pnpm (frontend), npm (backend) |

---

## Running It Yourself

```bash
# Backend
cd backend/backend
npm install
npm run dev          # API on http://localhost:3000

# Frontend
cd frontend/client
pnpm install
pnpm dev             # UI on http://localhost:4000

# API docs
open http://localhost:3000/api/docs
```

Default login: `admin` / `admin`

---

## What's Next

- **AI Chat Assistant** - Already built and integrated
- Docker Compose for one-command setup
- E2E tests with Playwright
- PostgreSQL support (the Sequelize config is ready, just needs the dialect switch)

---

Pivotal's Spring PetClinic taught a generation of Java developers how to build web apps. I hope this TypeScript version can do the same for the next generation - showing that you can take a proven domain model and rebuild it with modern tools, clean architecture, and attention to detail.

**Credits:** The original [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) was created by **Pivotal** (now VMware Tanzu) and is maintained by the Spring team. The domain model, database schema, and sample data in this project are derived from their work. This TypeScript rebuild is an independent reimplementation - all application code, UI design, authentication system, and AI features are original.

**The code is on GitHub.** Star it, fork it, break it, make it yours.

---

*Inspired by Pivotal's Spring PetClinic. Built with TypeScript, powered by curiosity.*

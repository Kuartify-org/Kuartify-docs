# Kuartify Enterprise Template Engine - Master Project Plan

This document is the unified source of truth for the Kuartify Template Engine. It outlines the architectural pillars, technology stack, and directory structure for our multi-tenant, JSON-driven platform.

---

## 1. Core Architectural Pillars

### A. The Universal Rendering Engine (The Interpreter)
- **Concept**: A single Next.js route (`app/[tenantDomain]/[...slug]/page.tsx`) acts as the "Brain." It is a 100% logic-only file that interprets JSON templates into HTML.
- **Dynamic Registry**: Uses `next/dynamic` to lazy-load only the components requested in the JSON.
- **Data Hydration**: Fetches page-level data once and shares it across the client/server tree using Request Memoization.

### B. Three-Tier Caching & Scaling
- **Layer 1 (Edge/CDN)**: Globally cached HTML via Incremental Static Regeneration (ISR).
- **Layer 2 (User-Scoped Redis)**: Personal data (Cart, Session) is cached using `{tenant_id}:{user_id}:{device}` keys.
- **Layer 3 (Smart Invalidation)**: Every `config.json` variable has a "Cache Invalidation Map" (e.g., `invalidate_cache: ["*"]` or `["home"]`) to ensure surgical precision during updates.

### C. The Standardized API Contract
- **Source of Truth**: All templates interact with a unified API library (`api.kuartify.com`).
- **Data Binding**: Supports visual mapping of props to **Global APIs**, **Page Data Pools**, and **Local Storage**.

---

## 2. Template Development Ecosystem

### A. Kuartify Studio (`studio.kuartify.com`)
- **Visual Web IDE**: A proprietary, white-labeled builder for Superadmins, powered by the **Puck Engine** (MIT).
- **The AI Pilot**: An intelligent "Vision-to-JSON" pipeline that synthesizes layouts from screenshots using our specific component registry.
- **The K-Vars System**: Manages template variables with **Mandatory vs. Optional** constraints. Sites are blocked from publishing if required `config.json` keys (e.g., Stripe Key) are missing.

### B. The K-Registry (Isolated Blueprint)
- **Strict Isolation Policy**: A dedicated workspace (`packages/k-registry`) containing pure React components (Radix + Framer Motion).
- **Zero Dependencies**: Registry components never depend on app-level code (Studio/Renderer), ensuring a clean, reusable library.

---

## 3. Technology Stack Summary
- **Framework**: Next.js 14+ (App Router, Server Components).
- **Builder Core**: Puck (React-native visual engine).
- **Animations**: Framer Motion (Scroll, Hover, Entry transitions).
- **Accessibility**: Radix UI (Primitives for unstyled logic).
- **Mocking**: MSW (Mock Service Worker) for the Studio's standalone designers.
- **Performance**: Redis (Upstash) + Cloudflare (Edge).

---

## 4. Monorepo Directory Structure Overview
Kuartify follows a modern monorepo structure (Turborepo) to keep the Studio, Engine, and Registry perfectly synced.

```text
kuartify-monorepo/
├── apps/
│   ├── studio/          (The Web IDE at studio.kuartify.com)
│   │   └── components/  (Studio-specific UI: Inspector, AI modals)
│   ├── renderer/        (The production Engine at [tenantDomain])
│   │   └── lib/         (Interpreter, Resolver, and Cache logic)
│   ├── dashboard/       (Main Kuartify management for Tenants)
│   ├── www/             (Kuartify Main Landing/Marketing Pages kuartify.com)
│   └── administrator/   (Kuartify Adminstrator Pages)
├── packages/
│   ├── k-registry/      (STRICTLY ISOLATED Template Components)
│   │   ├── atoms/       (Buttons, Inputs, Badges)
│   │   └── blocks/      (Hero, ProductGrid, Testimonials)
│   ├── config-ts/       (Shared TypeScript configurations)
│   └── eslint-config/   (Shared linting rules)
├── dump/                (Architectural Documentation)
├── package.json
└── turbo.json
```

---

## 5. Implementation Roadmap
1. [x] **Phase 1**: Architectural Design & Performance Logic.
2. [x] **Phase 2**: API Contract & Data Binding Specification.
3. [x] **Phase 3**: Template Development & Studio Architecture.
4. [ ] **Phase 4**: Final Integration & Live Data Connectivity.

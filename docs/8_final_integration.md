# Phase 4: Final Integration & Monorepo Scaffolding

This is the comprehensive, final guide for connecting the Kuartify architecture to the NestJS backend and live infrastructure. This document serves as the implementation blueprint for Phase 4 of the architectural plan.

---

## 1. Internal Technologies & Monorepo Setup
We will implement a **Turborepo Monorepo** for clean separation of concerns and professional maintenance.

- **Backend API**: **NestJS** (Centralized Auth, Database logic, and Redis cache management).
- **Storefront Engine**: Next.js (The `apps/renderer` interpreter).
- **Marketing Site**: Next.js (The `apps/www` homepage).
- **SuperAdmin Builder**: Next.js (The `apps/studio` Puck editor).
- **Internal Administrator**: Next.js (The `apps/administrator` platform management).
- **Component Registry**: `@kuartify/k-registry` (The standalone UI blueprint).

---

## 2. Directory Structure Overview
To ensure marketing, store rendering, and the Studio are isolated for clean maintenance, we use the following hierarchy:

```text
kuartify/
├── apps/
│   ├── www/             # Marketing & Landing (kuartify.com)
│   ├── renderer/        # Storefront Engine (Next.js Universal Interpreter)
│   ├── studio/          # SuperAdmin Builder (Next.js Puck)
│   ├── dashboard/       # Tenant Dashboard (Next.js)
│   └── administrator/   # Kuartify Staff Admin (Next.js)
├── packages/
│   ├── k-registry/      # STRICTLY ISOLATED Template Components (Radix + Motion)
│   ├── config-ts/       # Shared tooling configs
│   └── eslint-config/   # Shared linting rules
```

---

## 3. High-Speed Domain Mapping & Resolution
We handle apex domains (`business.com`), subdomains (`shop.business.com`), and kuartify subdomains (`business.kuartify.com`) using a single `middleware.ts`.

### Logic Flow:
1. **Extraction**: Middleware reads the `Host` header.
2. **Lookup**: Middleware checks an **Edge Cache (KV/Redis)** for the mapping.
3. **Failover**: If the cache is cold, it queries the **NestJS Backend**.
4. **Internal Rewrite**: Middleware performs an internal rewrite to `/[tenantId]/[slug]`.

---

## 4. The Rendering Engine (How JSON Becomes a Website)
### Step-by-Step Lifecycle

1. **Visitor** hits the domain.
2. **Middleware** rewrites to the correct internal tenant route.
3. **Interpreter** (`page.tsx`) fires:
   - Fetches **Mapper**, **Config**, and **Page JSON** from the NestJS Backend.
   - Core fetches are **ISR-cached** at the Edge.
4. **Validation**: The engine checks the **Mandatory K-Vars** defined in the schema.
5. **UI Assembly**: The engine maps JSON to `k-registry` components and generates static HTML.
6. **Hydration**: Browser receives HTML; Client Components (Cart, Auth, Animations) hydrate automatically.

---

## 5. Security & Private Data Fetching
To maintain extreme performance, the main site is server-rendered (Cached HTML), but private user data is handled via **Hybrid Rendering**:

- **Server-Side**: Renders the shell, product catalogs, and public content.
- **Client-Side**: Components like `<CartBadge />` or `<UserGreeting />` use `useEffect` to fetch private data from the NestJS API using secure session cookies/JWT.

---

## 6. Professional DNS & Verification Strategy
To maintain enterprise-grade stability and security, we follow the industry-standard "Hybrid Pointing" model.

### A. Hybrid DNS Pointing
- **For Subdomains** (`shop.client.com`): Users point a **CNAME** record to `proxy.kuartify.com`. (Allows us to change IPs without customer intervention).
- **For Root Domains** (`client.com`): Users point an **A-Record** directly to our static Anycast IP.

### B. Domain Verification (TXT Record)
Before any traffic is routed, the tenant must prove ownership.
- **Action**: Tenant adds a TXT record `kuartify-verify=[unique_hash]`.
- **Validation**: NestJS queries DNS via `dig` or a DNS library to verify the hash before enabling the store in the Resolver.

### C. Automated SSL (TLS Termination)
Every custom domain is served over `https`.
- **Strategy**: Leverage **Cloudflare for SaaS** (SSL for SaaS) or **Caddy Server** with On-Demand TLS.
- **Workflow**: When a domain is verified, our backend triggers the issuance of an SSL certificate automatically.

---

## 7. Verification & Final Handover
1. **Domain Resolution & Fallback Test**:
   - **Valid Store**: Verify `demo.kuartify.com` loads the template.
   - **Unregistered Host**: Verify an unmapped domain shows the professional "Under Development" page with the target owner-message.
2. **Isolation Audit**: Ensure `packages/k-registry` remains 100% independent.
3. **Performance Audit**: Verify 0ms database calls at origin for cached pages.

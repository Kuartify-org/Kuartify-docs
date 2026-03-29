# Enterprise Template Architecture Plan for Kuartify

To build a highly optimized, enterprise-grade template engine that supports thousands of tenants, we must leverage the full power of **Next.js App Router (Server Components + ISR + Edge Caching + Middleware)**.

Here is the deep-down architectural plan to handle customized templates, multi-tenant routing, RBAC, and extreme performance optimizations.

---

## 1. Domain Resolution & Routing (The Middleware Layer)

You cannot have a separate deployment for each business. Instead, we use a single Next.js application that handles all domains dynamically.

**How it works:**
- **`middleware.ts`**: Intercepts every incoming request to your platform.
- It parses the `Host` header (e.g., `grocery.kuartify.com` or `custom-hardware.com`).
- It maps the domain to a tenant identifier and performs a **rewrite** under the hood:
  `grocery.kuartify.com/products` → `app/[tenantDomain]/products/page.tsx`
- **Protected Routes**: The middleware checks for JWT/Authentication cookies. If the route (e.g., `/cart`, `/checkout`, `/admin`) is protected and the user is not authenticated, it immediately redirects them to `/login` *before* hitting the server.

---

## 2. The Rendering Engine (React Server Components + Registry)

Templates should not be separate apps. They are a combination of a global JSON configuration and a registry of React Components.

**The Strategy:**
- We build a standard library of UI components (`<Hero />`, `<Header />`, `<ProductGrid />`, `<Footer />`).
- The tenant's configuration (fetched from the backend) dictates which components render and what data they receive.

**Example Next.js Page (`app/[tenantDomain]/[...slug]/page.tsx`):**
```tsx
import { notFound } from "next/navigation";
import { fetchTenantConfig } from "@/lib/api";
import { ComponentRegistry } from "@/components/registry";

export default async function TenantPage({ params }) {
  // 1. Fetch JSON Config for this tenant and slug
  const config = await fetchTenantConfig(params.tenantDomain, params.slug);
  
  if (!config) return notFound();

  // 2. Render UI on the SERVER based on the JSON
  return (
    <div style={{ "--primary": config.theme.primaryColor }}>
      {config.sections.map((section, idx) => {
        const Component = ComponentRegistry[section.type];
        return <Component key={idx} {...section.props} />;
      })}
    </div>
  );
}
```

---

## 3. Extreme Optimization Strategy (ISR & The "Shopify Approach")

If 1,000 users visit a store's homepage, hitting the backend database 1,000 times to fetch the template JSON will crash the server.

**The Fix: Incremental Static Regeneration (ISR) + Next.js Data Cache**
- When Next.js runs `fetchTenantConfig`, we attach a cache rule:
  `fetch(url, { next: { revalidate: 3600, tags: [tenantDomain] } })`
- **What happens:**
  1. **User 1 visits:** Next.js securely calls your backend API, compiles the React components into HTML, and sends it to the browser. Next.js **saves (caches) this HTML at the Edge (CDN).**
  2. **Users 2 to 100,000 visit:** They bypass the database entirely. They get the pre-rendered HTML served instantly from the CDN cache. Zero API calls made.
  3. **The Store Owner updates their template:** You trigger a Next.js `revalidateTag(tenantDomain)` webhook from your backend. The cache is cleared, and the next visitor builds the new version.

---

## 4. Addressing Dynamic Interactivity & RBAC (Hybrid Rendering)

While the page layout (Header, Footer, Hero) can be fully cached via ISR, things like the Shopping Cart, User Profile, or Role-Based buttons (RBAC) are highly dynamic.

**The Solution: Partial Hydration & Client-Side Interactivity**
- **Server Components (Static/Cached):** The shell of the page, SEO meta tags, product grids, and template layouts are rendered on the server.
- **Client Components (Dynamic/Uncached):** 
  - An `<AddToCartButton productId="..." />` is a Client Component. It handles its own clicks and communicates with a dynamic API to update the cart state (e.g., using Zustand).
  - RBAC checks are done on the dynamic parts: The server can render the `Admin Edit Button` conditionally based on session cookies (which opts that tiny part out of the static cache), or the client hides it until the user auth state resolves.

---

## 5. Transition Effects & UI Polish

- You mentioned smoother transitions. With the App Router, we use `framer-motion` inside our Client Components.
- The `template.js` or `layout.js` file provided in Next.js can wrap pages in `<AnimatePresence>` for seamless route transitions that feel like a native app.
- Theme injection: We use CSS variables (e.g., `--color-primary`) injected at the root `<html style={{...}}>`. The components inherit these variables, meaning changing a theme color takes 0 compilation time—it's instantaneous.

---

## Summary of the Enterprise Architecture Flow

1. **Request enters:** `business.kuartify.com/shop`
2. **Middleware:** Checks auth (if required), rewrites to `/[domain]/shop`.
3. **Next.js Cache Check:** Is this page cached?
   - **Yes:** Serve instantly (0 API calls).
   - **No:** Proceed to step 4.
4. **Server Component Runs:** Fetches template JSON from Kuartify Backend API.
5. **Renderer Engine:** Maps JSON sections to React Components.
6. **Delivery:** HTML + CSS is sent to the client. Client-side JS "hydrates" interactive parts like the Cart and User Profile.

**Why this is Enterprise Grade:**
- **Zero-Latency Feel:** Pages load instantly from the CDN.
- **Backend Protected:** Caching prevents DDoS and database bottlenecks.
- **High SEO:** Search engines see fully rendered HTML, not blank CSR screens.
- **One Codebase:** You never deploy an app when a new store signs up. You just create an entry in the backend database.

---

## 6. Template Usage & Configuration Architecture

### A. Centralized Referencing & Auto-Publishing
Store Owners do not "clone" templates into their databases. They securely reference a single, centralized Template ID maintained by Kuartify Superadmins.
- When a Superadmin publishes a new version of `temp_123` (from Draft to Published), all tenants mapping to `temp_123` automatically fetch the newly published version immediately after the Edge cache invalidates. Tenants do not map to specific version strings, they only map to the Template ID.

### B. Config-Driven Customization (`config.json`)
Store Owners customize their digital identity strictly within `config.json` via the Kuartify Dashboard. This file holds:
1. **Thematic Tokens**: Primary colors, typography.
2. **Assets**: Logos, Hero images.
3. **Content Overrides**: Localized text (e.g., "About Us").
4. **Platform Integrations**: Server secrets (e.g., Stripe API Keys, Analytics IDs).

**Smart Cache Invalidation:** Every key in the `config.json` maps strictly to an invalidation array. 
- Changing `theme.primary_color` maps to `["*"]` (invalidating the entire global cache).
- Changing `content.about_us.hero_image` maps to `["aboutUs"]`, invalidating only that specific Edge route and saving massive backend database load.

### C. Deployment Statuses & A/B Testing
A tenant can only have **1 active/live website** running a template. However, they can safely manipulate multiple configurations:
1. **Drafts**: Owners can maintain multiple `config.json` drafts for the active template to preview changes safely without affecting live traffic.
2. **A/B Testing**: Owners can A/B test their Live `config.json` against one Draft `config.json`. 
   - *Constraint*: A/B Testing is restricted strictly to the tenant's Kuartify subdomain (e.g., `business.kuartify.com`). It dynamically splits traffic utilizing Next.js Middleware and cookies, safely measuring conversion metrics before pushing the experiment Live.

# Kuartify Template Rendering Engine Specification

This document details the complete workflow, guidelines, and purpose of the Kuartify Template Rendering Engine. 

*(Note: Edge Caching and Redis strategies are detailed in separate documentation).*

---

## 1. Engine Purpose
The Engine (`app/[tenantDomain]/[...slug]/page.tsx`) acts as the singular interpreter for all tenant storefronts. It never dictates design; it maps JSON layout definitions to React Server Components and orchestrates data hydration securely.

## 2. Page Resolution & Hard 404 Handling
When a user visits a parameterized route (e.g., `business.com/products/123`), the Engine cross-references `mapper.json`.

- **Page Does Not Exist**:
  - The Engine returns a **hard HTTP 404 status code** to the browser (`return new Response(html, { status: 404 })`).
  - It renders the `404.json` UI.
  - *Why?* This prevents Search Engines from indexing broken pages and prevents the CDN from caching a 404 page as a "valid" `200 OK` response.

## 3. The Two-Step Rendering Algorithm (Backend Authority)

The template JSON *proposes* data fetching rules, but the **Backend is the ultimate authority**.

### Step 1: Server-Side Assembly (Public Data)
- The Engine reads the JSON components. If a component requests an API call, the Engine checks a backend utility: `api.isPublic(endpoint)`.
- If the backend confirms the API is public (e.g., Product Catalog), the Engine fetches the data immediately on the server, assembles the UI, and outputs static HTML.

### Step 2: Client-Side Fetching (Private/Session Data)
- If the backend flags the API as private (e.g., User Orders), the server-side Engine **refuses** to fetch it, regardless of what the template JSON requested.
- The Engine outputs an empty container (Skeleton).
- The user's browser securely makes the API request using their session token.

## 4. Partial Hydration Strategy (Priority Tiers)
To ensure the UX feels instantaneous, the Engine processes UI components in structured tiers:

- **Tier 1 (Critical - Server Rendered)**: 
  - Navbar, Hero Image, Key Product Info.
  - Generates immediately in HTML format.
- **Tier 2 (Deferred - Client Hydrated)**: 
  - Reviews, Recommendations, Shopping Cart state.
  - Fetched and mounted just after the primary shell loads completely.
- **Tier 3 (Lazy - On Demand)**: 
  - Heavy Analytics widgets, off-screen Admin edit toggles.
  - Loaded completely asynchronously or only when scrolled into view using `React.lazy`.

## 5. Defense-in-Depth RBAC
- **UI Level**: The Engine parses `"visibleFor": ["admin"]` in the template JSON and strips the component from the HTML if the user lacks the role.
- **API Level**: The Engine implicitly relies on Kuartify's backend APIs to enforce real authorization. Even if a malicious user bypasses the UI role-check to force a client-side fetch, the API will strictly reject the unauthorized request.

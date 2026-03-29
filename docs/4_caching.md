# Kuartify Caching Architecture

To achieve Shopify/Wix level performance without overloading the Kuartify database, the system employs a highly targeted **Three-Tier Caching Strategy**. 

---

## Layer 1: The HTML Cache (Next.js ISR)
This layer handles the **Static Page Shells** (the UI components, themes, and public data like product catalogs).

- **How it works:** When a visitor hits a tenant's page, Next.js generates the HTML and caches it. The next visitors receive this HTML instantly.
- **Hosting & Cost (Free/Zero-Config):** 
  - If you deploy on **Vercel** (the creators of Next.js), this happens *automatically* at the global Edge CDN level with zero code configuration on their free tier.
  - If you self-host on a cheap **VPS (like DigitalOcean/AWS EC2)**, Next.js automatically caches it beautifully to your local server disk. You can then put a **Free Cloudflare proxy** in front of your server, and Cloudflare will distribute that cached HTML globally for you at no cost.
- **The Rule:** This cache is strictly **Global**. It must never contain data specific to a single user's session.
- **Tagging Strategy:** Cached pages must be tagged granularly to allow targeted invalidation. (e.g., `tenant-XYZ`, `tenant-XYZ-page-home`, `template-123-orders`).

---

## Layer 2: The User-Scoped Data Cache (Redis)
This layer protects the Kuartify backend from traffic spikes caused by private Client-Side Data Fetching.

- **The Problem:** If 10,000 users load the `/orders` page, they instantly load the Edge HTML. But their browsers immediately fire 10,000 requests for their private data.
- **The Solution (Redis):** 
  - Instead of hitting the primary database, the API hits a fast Redis in-memory cache.
  - **The Cache Key Format:** To prevent data collisions across platforms and organizations, the Redis keys follow a strict format: 
    `{tenant_id}:{user_id}:{device_type}:{page_context}`
    - `tenant_id`: Ensures data isolation between different storefronts (e.g., `shop-123`).
    - `user_id`: Ensures data isolation between different authenticated customers (e.g., `user-999`).
    - `device_type`: Prepares the architecture for mobile apps vs. web (e.g., `web` or `mobile`).
    - `page_context`: Identifies the specific data payload being cached (e.g., `MyOrders` or `Profile`).
  - **Example Key:** `shop-123:user-999:web:MyOrders`

---

## Layer 3: Smart Cache Invalidation (The Targeted Purge)
A cache is only useful if it knows precisely *what* and *when* to clear. We use a **Smart Invalidation Map** to prevent unnecessary purges.

### 1. Store Owner Action (The `config.json`)
A store owner only has access to modify their store's `config.json` (theme colors, logo, settings).
- **Action:** Store Owner updates Theme Colors.
- **Invalidation:** The API must instantly invalidate the **FULL website cache** (CDN and Redis) strictly for that specific `tenant-XYZ`.

### 2. Superadmin Action (The Template Architecture)
A template is a collection of files (`mapper.json`, `meta.json`, `about.json`, `orders.json`). Only Superadmins can update these structural files.
- **Action:** Superadmin updates `orders.json` within Template 123.
- **Invalidation:** The API purges the CDN and Redis caches for the `orders` page across **all tenants** globally who are utilizing Template 123.

### 3. API-Driven Targeted Actions
When typical data payloads are modified, we use an Invalidation Map defined by the template to precisely target the purge. 
*Note: An "action" here typically maps to a specific Backend API request (e.g., `POST /api/transaction`), but it can also represent a background worker event or an external Webhook (like Stripe confirming a payment).*

- **Example A (Public Data Change):** 
  - **Action:** Admin adds a new product or category via API.
  - **Invalidation Map:** `{ "action": "ADD_PRODUCT", "targets": ["home", "catalogue"] }`
  - **Result:** Only the `/` and `/catalogue` pages in the Edge CDN for that specific tenant are purged. The rest of the site remains untouched in the cache.
  
- **Example B (Private User Data Change):** 
  - **Action:** A customer successfully checks out and places an order (`POST /api/transaction`).
  - **Invalidation Map:** `{ "action": "PLACE_ORDER", "targets": ["MyOrders"], "scope": "user_specific" }`
  - **Result:** Instead of clearing everyone's cache, the API targets Redis accurately using the full schema: `redis.del('tenant-XYZ:user_123:web:MyOrders')` (and mobile if applicable). The global Edge Cache remains completely untouched.

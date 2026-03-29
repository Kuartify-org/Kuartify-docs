# Kuartify Template Development Overview

This document outlines the high-level lifecycle and architecture for creating, managing, and distributing templates across the Kuartify platform.

---

## 1. The Core Vision
Kuartify replaces traditional "code-based" theme development with a centralized **Visual Construction** model. Instead of writing Liquid (Shopify) or custom Next.js code for every tenant, templates are built as standardized JSON packages.

- **The Creator**: Superadmins and authorized Internal Architects.
- **The Workspace**: `studio.kuartify.com` (A centralized internal application).
- **The Engine**: A universal, JSON-driven Rendering Engine that interprets these templates at production scale.

---

## 2. Template Anatomy (What is a Template?)
A Template is a modular package containing the following core files:
- **`mapper.json`**: The routing map. It defines which URL paths (e.g., `/products`, `/about`) map to which page layout files.
- **`meta.json`**: Template identity, versioning metadata, and preview assets.
- **`pages/*.json`**: Individual page layout definitions (e.g., `home.json`, `product_detail.json`).
- **`config_schema.json`**: The "Contract" for Store Owners. It defines exactly which fields (colors, images, texts) a tenant is allowed to customize.
- **`registry_contract.json`**: A serialized manifest of all components available to that template from the global K-Registry.

---

## 3. The Custom "K-Registry"
Kuartify utilizes a proprietary component library built for 100% parity between the Studio and Production.
- **Foundation**: Built on top of **Radix UI** primitives for industrial-grade accessibility and behavior.
- **Strategy**: A "Hybrid Proprietary" model using a **Strictly Isolated** library.
- **Benefit**: This enables **Vision-to-JSON** AI generation, as the AI has a precise list of components to work with.

---

## 4. Architectural Decision: The Monorepo
To ensure professional-grade maintenance and 100% visual parity, Kuartify utilizes a **Turborepo Monorepo** architecture. 
- **The `packages/k-registry` Workspace**: This is the **exclusive, standalone home** for all template components. It must never depend on external app code.
- **Atomic Refactoring**: A single change to a component’s code updates the Studio’s builder UI and the production storefront simultaneously.
- **Separation of Concerns**: `apps/studio` and `apps/renderer` are consumers, while `packages/k-registry` is the pure source of truth.

---

## 5. The Development Lifecycle (How it works)

### Step 1: Studio Initialization
The architect initializes a project in `studio.kuartify.com`. The Studio environment provides a **Headless Sandbox** with mock API data, allowing development without touching production databases.

### Step 2: Visual Construction & AI Pilot
The architect assembles pages by dragging components from the K-Registry. 
- **AI Pilot**: Allows the architect to upload a screenshot or Figma URL. The AI analyzes the layout and automatically populates the canvas with a matching JSON structure using Kuartify-native components.

### Step 3: Data Binding 
Using a visual **Data Connector**, the architect binds component props to page-level API responses. This supports both **Static Binding** (single values) and **Dynamic Repeaters** (looping through product or order lists).

### Step 4: Schema Exposure
The architect selects specific props (e.g., "Banner Image") and toggles the **"Exposed to Tenant"** flag. This automatically builds the `config_schema.json`.
- **Mandatory Flag**: Architects can mark fields as `required: true`. The platform will **block** a tenant from publishing until these fields are filled.

### Step 5: Validation & Global Deployment
The Studio runs a global audit (schema checks, accessibility audits). Once passed, the template is **Published** as a new version. Every tenant referencing that Template ID inherits the update automatically.

# Detailed Plan - Kuartify Studio Architecture

This document provides a low-level deep dive into the internal mechanics of `studio.kuartify.com`, the centralized Web IDE for Kuartify Template Architects.

---

## 1. Internal Technologies & Libraries
The Studio is a professional-grade builder application living within the **Kuartify Monorepo**.

- **Builder Engine**: **Puck** (The visual canvas and drag-and-drop core).
- **Animations**: **Framer Motion** (Used within `@kuartify/k-registry` components to handle entry, hover, and scroll animations).
- **Mocking (MSW)**: **Mock Service Worker** intercepts network requests at the browser level. This allows the architect to see "Real" looking products and orders in the preview without needing a live backend running.
- **Component Primitives**: **Radix UI** (For accessible, unstyled UI logic).

---

## 2. Directory Structure & Strict Isolation
To ensure professional-grade maintenance, the Studio follows a **Strict Isolation Policy** for template-level components.

### The `packages/k-registry` Workspace
- **Purpose**: This is the **exclusive, isolated directory** for all drag-and-drop components used in templates.
- **Zero External Dependencies**: Components in this directory **must never** depend on any code from `apps/studio` or `apps/renderer`. They are entirely self-contained.
- **Dual Consumption**: Both the Studio (for building) and the Rendering Engine (for viewing) consume this package as their single source of truth.

### The App-Level Separation
- **`apps/studio/components/studio-ui`**: Designer-specific tools (e.g., Prop Inspector, AI Pilot modal, API binding dropdowns). This prevents "Builder logic" from leaking into the production Engine.
- **`apps/renderer`**: The tenant-facing interpreter (lightweight and optimized).
- **`apps/dashboard`**: The main Kuartify management app, which remains entirely separate from the Template/Registry ecosystem.

---

## 3. Sample `config.json` Schema (The Template .env)
Before a template is published, the Studio generates the `config_schema.json`. Below is a sample showcasing validation, UI types, and performance maps:

```json
{
  "theme_settings": {
    "primary_color": {
      "type": "color",
      "required": true,
      "default": "#a38bde",
      "invalidate_cache": ["*"], // Global purge on brand color change
      "label": "Brand Primary Color"
    },
    "layout_style": {
      "type": "enum",
      "required": true,
      "options": ["Modern", "Classic", "Minimal"],
      "invalidate_cache": ["home", "products"],
      "label": "Global Theme Style"
    }
  },
  "integrations": {
    "stripe_public_key": {
      "type": "key",
      "required": true, // Block publishing if missing
      "label": "Stripe Public Key"
    },
    "custom_announcement": {
      "type": "text",
      "required": false, // Optional
      "invalidate_cache": ["home"],
      "label": "Homepage Announcement"
    }
  }
}
```

---

## 4. Visual Canvas & Animation Handling
The **Puck Engine** renders components inside an isolated `iframe` to prevent CSS leaking.
- **Hover/Scroll Effects**: Handled by **Framer Motion**.
- **Edit Mode Awareness**: Components receive a `puck.isEditing` prop. When true, high-intensity animations can be paused to keep the builder experience smooth and responsive.

---

## 5. The Data Binding UI (Visual Logic)
This is where the architect connects UI elements to data sources.

- **Source Selector**: Choose between Global API, Page Data Pool, or **Local Storage** (e.g., `email`, `username`, `cart_id`). 
- **Auto-Mapping**: The Studio parses the mock response tree from `api_declarations.md`. Clicking a key directly binds it to a component's prop.
- **Dynamic Repeaters**: For containers (like a Product Grid), the architect selects an array from the API response to "iterate" over.

---

## 6. The AI Pilot Pipeline (Vision-to-JSON)
The intelligence layer that speeds up template creation by 10x:
1. **Context Injection**: AI receives the `registry_contract.json` (The Manifesto).
2. **Synthesis**: AI identifies layout patterns in a screenshot and maps them to our proprietary components.
3. **Drafting**: AI generates a valid `page.json` and injects it directly into the Studio's Puck canvas.

---

## 7. No-Code Constraints & Security
- **100% No-Code**: Superadmins cannot write custom JavaScript blocks. 
- **Action Components**: All complex logic (e.g., "Add to Cart", "Stripe Checkout") is provided via pre-built components that contain the necessary logic internally. This ensures extreme stability and security.

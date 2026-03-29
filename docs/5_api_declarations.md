# Kuartify API Declarations & Engine Contract

This document maps the exact JSON response that will be returned by `GET api.kuartify.com/api`. This single JSON structure acts as the source of truth for all template-usable APIs globally across the Kuartify platform.

---

## 1. Why Do We Need This Centralized API Structure?

In traditional web development, a frontend developer simply looks at documentation, writes a custom `fetch()` call for a specific page, and handles the data. 

**Kuartify is not a traditional web app.** It is a dynamic JSON-driven rendering engine. The single Engine (`[...slug]/page.tsx`) must be able to securely construct any page, for any tenant, using any API, without writing custom hard-coded rules for every possible scenario. 

We need a **Centralized API Contract** because:
1. **The Engine is Blind**: The engine doesn't inherently know if an API needs `x-tenant-id` or an `Authorization` header. It must read a strict definition to safely construct the fetch request.
2. **Preventing Edge Cache Leaks**: As established in our caching architecture, the Engine must know immediately if a template component attempting to use an API is allowed to cache it (`"allowCache": false`). This dictionary tells the Engine the absolute rules.
3. **Structured Error Handling**: A unified and overrideable `common_responses` dictionary automatically handles edge cases globally. Developers don't need to manually declare error states for every component. However, if a template *requires* an override, the developer simply uses the status code in the component schema and rewrites their custom error message to be shown on the UI.

---

## 2. How Will This Structure Be Used?

This JSON dictionary interacts with three distinct actors:

### A. The Superadmin (Template Integration Phase)
When a Kuartify Superadmin creates a new template (`mapper.json`), they use this API library to know exactly what data the templates can request.
- *Scenario*: The Superadmin adds a `<LoginForm />` building block to the template. They query `api.kuartify.com/api`, see that `POST /api/login` expects an `email` and `password` body, and map those component input fields directly to the API's required properties.

### B. The Store Owner (Configuration Phase)
The Store Owner does not edit this code, but they are bound by its rules. If the API dictionary marks an endpoint as requiring `x-tenant-id`, the Engine safely extracts that ID from the Owner's database configuration automatically when they publish their theme.

### C. The Rendering Engine (Execution Phase)
When the user clicks "Checkout", the JSON template fires an action (e.g., `"submitTo": "/api/orders"`). 
1. The Engine looks up `/api/orders` in this unified dictionary.
2. The dictionary dictates: *"Method is GET, Authorization header is required, allowCache is false."*
3. The Engine dynamically constructs a secure CSR (Client-Side Rendering) request fetching the active user's cookie, executes the call, and maps the `response_success` data directly back into the UI state.

---

## 3. The Unified API Library JSON

```json
{
  "common_responses": {
    "response_notFound": {
      "status_code": 404,
      "message": "Not Found"
    },
    "response_forbidden": {
      "status_code": 403,
      "message": "Forbidden access to this resource."
    },
    "response_unauthorized": {
      "status_code": 401,
      "message": "Unauthorized request. Token missing or invalid."
    },
    "response_serverError": {
      "status_code": 500,
      "message": "Internal Server Error."
    }
  },
  "available_apis": [
    {
      "method": "GET",
      "route": "/api/orders",
      "description": "Fetches a paginated list of all active and past orders placed by the currently authenticated user.",
      "allowCache": false,
      "headers": {
        "Authorization": "Bearer <token> (Required)",
        "x-tenant-id": "<string> (Required)"
      },
      "queryParams": {
        "page": "number (Optional, default 1)",
        "limit": "number (Optional, default 10)"
      },
      "bodyStructure": null,
      "response_success": {
        "status_code": 200,
        "status": "success",
        "data": {
          "pagination": {
            "current_page": 1,
            "total_pages": 5,
            "total_items": 42
          },
          "orders": [
            {
              "id": "ord_8f72h19",
              "created_at": "2026-03-28T10:00:00Z",
              "status": "processing",
              "total": 129.99,
              "currency": "USD",
              "items": [
                {
                  "product_id": "prod_1",
                  "name": "Kuartify Premium Hoodie",
                  "quantity": 1,
                  "price": 129.99
                }
              ],
              "tracking_number": null
            },
            {
              "id": "ord_2b49f01",
              "created_at": "2026-02-15T14:30:00Z",
              "status": "delivered",
              "total": 45.00,
              "currency": "USD",
              "items": [
                {
                  "product_id": "prod_8",
                  "name": "Minimalist Desk Mat",
                  "quantity": 1,
                  "price": 45.00
                }
              ],
              "tracking_number": "TRK9982736152"
            }
          ]
        }
      }
    },
    {
      "method": "POST",
      "route": "/api/login",
      "description": "Authenticates a storefront customer and issues a secure JWT session token.",
      "allowCache": false,
      "headers": {
        "Content-Type": "application/json (Required)",
        "x-tenant-id": "<string> (Required)"
      },
      "queryParams": null,
      "bodyStructure": {
        "email": "customer@example.com",
        "password": "securepassword123"
      },
      "response_success": {
        "status_code": 200,
        "status": "success",
        "data": {
          "token": "eyJhbGciOiJIUzI1NiIsIn... (full JWT)",
          "expires_in": 3600,
          "user": {
            "id": "usr_94812",
            "email": "customer@example.com",
            "first_name": "Hamza",
            "last_name": "Husain",
            "role": "customer"
          }
        }
      }
    }
  ]
}
```

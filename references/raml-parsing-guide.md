# RAML 1.0 Parsing Guide

Use this guide to extract the Semantic Model from RAML 1.0 specification files (YAML).

---

## 1. Format Detection

Confirmed RAML 1.0 if first line is exactly:
```
#%RAML 1.0
```

---

## 2. Metadata Extraction

```yaml
#%RAML 1.0
title: Order Service API          → app title, class naming, README
version: v1                       → used in application.yml + base path
baseUri: https://api.example.com/{version}   → base path (resolve {version})
mediaType: application/json       → assume JSON for all request/response bodies
description: ...                  → README overview
```

**Naming rule:** Convert `title` to PascalCase for class names, kebab-case for artifact name.
Resolve `baseUri` by substituting `{version}` with the `version` value.

---

## 3. Endpoint Extraction

RAML uses nested resource syntax:

```yaml
/orders:
  description: Order management
  get:                                  → HTTP GET → @GetMapping("/orders")
    displayName: List Orders            → Javadoc + operationId candidate
    description: Returns all orders
    queryParameters:
      page:
        type: integer                   → @RequestParam Integer page
        required: false
        default: 0
      size:
        type: integer
        required: false
        default: 20
    responses:
      200:
        body:
          application/json:
            type: OrderPage             → response DTO class
  post:                                 → HTTP POST → @PostMapping("/orders")
    displayName: Create Order
    body:
      application/json:
        type: CreateOrderRequest        → request body DTO
    responses:
      201:
        body:
          application/json:
            type: OrderResponse
      400:
        body:
          application/json:
            type: ErrorResponse

  /{orderId}:                           → nested path param → "/orders/{orderId}"
    uriParameters:
      orderId:
        type: string
        (format): uuid                  → UUID Java type
    get:
      displayName: Get Order By ID
      responses:
        200:
          body:
            application/json:
              type: OrderResponse
        404:
          body:
            application/json:
              type: ErrorResponse
```

**Path construction:** Concatenate parent paths with child paths.
`/orders` + `/{orderId}` → `/orders/{orderId}`

**operationId generation:** If `displayName` present, convert to camelCase.
Otherwise generate from method + path: `GET /orders/{orderId}` → `getOrderByOrderId`

**Controller grouping:** Group by top-level resource (first path segment).
`/orders` and `/orders/{orderId}` → both go into `OrderController`.

---

## 4. Type / Schema Extraction

RAML types are defined in the `types` section or inline:

```yaml
types:
  CreateOrderRequest:
    type: object
    properties:
      customerId:
        type: string
        (format): uuid               → UUID Java type
        required: true               → @NotNull
      items:
        type: OrderItem[]            → List<OrderItem>
        minItems: 1                  → @Size(min=1)
        required: true
      notes:
        type: string
        maxLength: 500               → @Size(max=500)
        required: false              → no @NotNull

  OrderItem:
    type: object
    properties:
      productId:
        type: string
        required: true
      quantity:
        type: integer
        minimum: 1                   → @Min(1)
        maximum: 100                 → @Max(100)
        required: true

  OrderResponse:
    type: object
    properties:
      orderId:
        type: string
        (format): uuid
      status:
        type: string
        enum: [PENDING, CONFIRMED, SHIPPED, DELIVERED]   → Java enum
      createdAt:
        type: datetime               → LocalDateTime
```

**Inline types:** If a type is defined inline (not in `types` section), extract it,
name it based on the operationId + "Request"/"Response" suffix, and add to model list.

---

## 5. RAML → Java Type Mapping

| RAML type | Annotation/Format | Java type |
|---|---|---|
| string | (none) | String |
| string | (format): uuid | UUID |
| string | (format): date | LocalDate |
| datetime | (none) | LocalDateTime |
| date-only | (none) | LocalDate |
| time-only | (none) | LocalTime |
| integer | (none) | Integer |
| number | (none) | Double |
| boolean | (none) | Boolean |
| array | (none) | List<{itemType}> |
| {Type}[] | (none) | List<{Type}> |
| object | (none) | Map<String, Object> or dedicated class |
| enum values | (none) | Java enum |
| nil | (none) | void / Void |

**Array shorthand:** `OrderItem[]` → `List<OrderItem>`

---

## 6. Security Scheme Extraction

```yaml
securitySchemes:
  oauth_2_0:
    type: OAuth 2.0
    settings:
      accessTokenUri: https://auth.example.com/token   → note for application.yml
      authorizationGrants: [client_credentials]

securedBy: [oauth_2_0]             → applied globally OR per endpoint
```

Always generate `SecurityConfig` with JWT resource server regardless of spec security scheme
(per skill configuration). Use `accessTokenUri` as placeholder in `application.yml`.

Per-endpoint security:
```yaml
/orders:
  get:
    securedBy: [oauth_2_0]         → this endpoint requires auth (all do per skill config)
    securedBy: null                → public endpoint — note to user but still secure per config
```

---

## 7. Traits & Resource Types (RAML reuse patterns)

```yaml
traits:
  paginated:
    queryParameters:
      page:
        type: integer
        default: 0
      size:
        type: integer
        default: 20

resourceTypes:
  collection:
    get:
      is: [paginated]
      responses:
        200:
          body:
            type: <<resourcePathName | pluralize>>Page
```

**Resolution:** Expand all `is:` trait references and `type:` resource type references
inline before building the semantic model. Treat expanded result as if written explicitly.

---

## 8. Libraries and Includes

```yaml
uses:
  common: ./common-types.raml      → external library; ask user to upload if not present

!include ./fragments/order.raml   → inline include; ask user to upload if not present
```

If referenced files are not uploaded, flag them to the user before proceeding:
> "Your spec references `./common-types.raml` which was not uploaded.
> Please upload it so I can resolve the shared types."

---

## 9. Error Response Extraction

```yaml
responses:
  400:
    body:
      application/json:
        type: ErrorResponse        → map to MethodArgumentNotValidException handler
  404:
    body:
      application/json:
        type: ErrorResponse        → map to domain NotFoundException
  500:
    body:
      application/json:
        type: ErrorResponse        → map to generic server error handler
```

Same mapping to `GlobalExceptionHandler` as OAS guide section 7.

---

## 10. Spec Validation Checks

Before proceeding to code generation, flag these issues to the user:
- Missing `displayName` on endpoint → Claude generates operationId from method + path
- Type used but not defined in `types` section and not a primitive → warn, generate placeholder DTO
- External `!include` or `uses` files not uploaded → stop and request files
- Endpoint with no success response → warn, generate `ResponseEntity<Object>`
- `required` not specified on property → default to `false` (optional field, no @NotNull)
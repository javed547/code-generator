# OAS 3.x Parsing Guide

Use this guide to extract the Semantic Model from OpenAPI 3.x specification files (YAML or JSON).

---

## 1. Format Detection

Confirmed OAS 3.x if file contains:
```yaml
openapi: "3.0.x"   # or 3.1.x
info:
  title: ...
```

---

## 2. Metadata Extraction

```yaml
info:
  title: Order Service API        → app title, used in README + class naming
  version: 1.0.0                  → used in application.yml + README
  description: ...                → used in README overview
servers:
  - url: https://api.example.com/v1   → base path for RestClient baseUrl (producer)
                                       → base path for @RequestMapping (consumer)
```

**Naming rule:** Convert `info.title` to PascalCase for class names, kebab-case for artifact name.
Example: "Order Service API" → `OrderServiceApi` (class), `order-service-api` (artifact)

---

## 3. Endpoint Extraction

For each entry under `paths`:

```yaml
paths:
  /orders/{orderId}:              → path (keep as-is for @RequestMapping)
    get:                          → HTTP method → @GetMapping
      operationId: getOrderById   → use as Java method name (camelCase)
      summary: ...                → Javadoc comment
      parameters:
        - name: orderId           → @PathVariable (if in: path)
          in: path                → @RequestParam (if in: query)
          required: true          → validation: @NotNull / @NotBlank
          schema:
            type: string          → Java type (see Type Mapping below)
            format: uuid          → UUID
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'  → request DTO class
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'    → response DTO class
        '400':
          $ref: '#/components/responses/BadRequest'           → exception handler
        '404':
          description: Order not found                        → domain exception
      security:
        - bearerAuth: []          → confirm JWT security on this endpoint
      tags:
        - Orders                  → controller class grouping
```

**Controller grouping:** Group endpoints by their first `tags` entry. One controller per tag.
If no tags, group by first path segment (e.g. `/orders/*` → `OrderController`).

---

## 4. Schema Extraction

For each schema under `components/schemas`:

```yaml
components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [customerId, items]       → @NotNull on these fields
      properties:
        customerId:
          type: string
          format: uuid                    → UUID type
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'   → List<OrderItem>
          minItems: 1                     → @Size(min=1)
        notes:
          type: string
          maxLength: 500                  → @Size(max=500)
          nullable: true                  → no @NotNull

    OrderItem:
      type: object
      properties:
        productId:
          type: string
        quantity:
          type: integer
          minimum: 1                      → @Min(1)
          maximum: 100                    → @Max(100)
```

---

## 5. OAS → Java Type Mapping

| OAS type | OAS format | Java type |
|---|---|---|
| string | (none) | String |
| string | uuid | UUID |
| string | date | LocalDate |
| string | date-time | LocalDateTime |
| string | email | String + @Email |
| string | uri | String |
| integer | (none) | Integer |
| integer | int32 | Integer |
| integer | int64 | Long |
| number | float | Float |
| number | double | Double |
| boolean | (none) | Boolean |
| array | (none) | List<{itemType}> |
| object | (none) | Map<String, Object> or dedicated class |
| $ref | (none) | Referenced class name |

**Nullability:** If `nullable: true` OR field not in `required` list → field is not annotated with `@NotNull`.

---

## 6. Security Scheme Extraction

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT             → confirms JWT; map to Spring Security JWT resource server
    oauth2:
      type: oauth2
      flows:
        clientCredentials:
          tokenUrl: https://auth.example.com/token   → note for application.yml
```

Always generate `SecurityConfig` with JWT resource server regardless of what spec says
(per skill configuration). Use spec token URLs as placeholder values in `application.yml`.

---

## 7. Error Response Extraction

```yaml
components:
  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    NotFound:
      description: Resource not found
```

Map to `GlobalExceptionHandler` handler methods:
- 400 → `MethodArgumentNotValidException`, `HttpClientErrorException.BadRequest`
- 401 → `HttpClientErrorException.Unauthorized`
- 403 → `HttpClientErrorException.Forbidden`
- 404 → `HttpClientErrorException.NotFound` + domain-specific `{Resource}NotFoundException`
- 500 → `HttpServerErrorException`

---

## 8. $ref Resolution

Always resolve `$ref` before building the semantic model:
- `#/components/schemas/Foo` → inline schema named `Foo`
- `./other-file.yaml#/components/schemas/Bar` → note as external ref; ask user if file not uploaded
- Circular refs → break cycle by using the class name as a forward reference

---

## 9. Spec Validation Checks

Before proceeding to code generation, flag these issues to the user:
- Missing `operationId` on any endpoint → Claude generates one from method + path
- Schema with no `type` → treat as `object`, generate dedicated DTO class
- Response with no schema → return `ResponseEntity<Void>`
- Endpoint with no success response defined → warn user, generate `ResponseEntity<Object>`
- Ambiguous path param types → default to `String`
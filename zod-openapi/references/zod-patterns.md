# Zod Patterns for API Documentation

Patterns for writing Zod schemas that produce correct, well-annotated OpenAPI output using `z.toJSONSchema({ target: "openapi-3.0" })` (Zod v4 native).

## `z.meta()` — The Documentation Hook

Every Zod type supports `.meta()`. Fields map directly to OpenAPI JSON Schema properties:

```ts
z.string().meta({
  title: "PayTag",
  description: "Short alphanumeric user handle. Used for wallet transfers.",
  example: "jane42",
  default: undefined,
  deprecated: false,
})
```

| `.meta()` key | OpenAPI output | When to use |
|---|---|---|
| `title` | `title` | Top-level named schemas only |
| `description` | `description` | Fields that need clarification |
| `example` | `example` | Fields or schemas shown in Scalar UI |
| `default` | `default` | Fields with well-known defaults |
| `deprecated` | `deprecated: true` | Fields being phased out |

---

## Primitive Patterns

### Strings

```ts
// Simple required string
z.string().min(1)

// Enum
z.enum(["active", "inactive", "pending"]).meta({ example: "active" })

// Email
z.string().email().meta({ example: "jane@example.com" })

// UUID
z.string().uuid().meta({ example: "550e8400-e29b-41d4-a716-446655440000" })

// ISO datetime
z.string().datetime().meta({ example: "2026-01-15T10:00:00.000Z" })

// Optional with trim
z.string().trim().optional()

// Optional with empty-string coercion (for query params)
z.preprocess(
  (v) => (typeof v === "string" && v.trim() === "" ? undefined : v),
  z.string().optional()
)
```

### Numbers

```ts
// Required integer
z.number().int().min(1)

// Positive float
z.number().positive()

// Coerced from query string (REQUIRED for query params)
z.coerce.number().int().min(1).optional()

// Range-bounded
z.coerce.number().int().min(1).max(100).optional().meta({ example: 20 })
```

### Booleans

```ts
// Coerced from query string ("true"/"false")
z.preprocess(
  (v) => v === "true" ? true : v === "false" ? false : v,
  z.boolean().optional()
)
```

---

## Object Patterns

### Basic request body

```ts
const createUserSchema = {
  body: z.object({
    email: z.string().email().meta({ example: "jane@example.com" }),
    password: z.string().min(8),
    firstName: z.string().trim().optional().meta({ example: "Jane" }),
    role: z.enum(["user", "admin"]).optional().meta({ example: "user" }),
  }),
};
```

### Partial update (PATCH body — all fields optional)

```ts
const updateUserSchema = {
  body: z.object({
    firstName: z.string().trim().optional(),
    lastName: z.string().trim().optional(),
    picture: z.string().url().optional(),
  }).refine(
    (data) => Object.values(data).some((v) => v !== undefined),
    { message: "At least one field is required" }
  ),
};
```

### Nested objects

```ts
z.object({
  address: z.object({
    street: z.string(),
    city: z.string(),
    country: z.string().length(2).meta({ example: "NG" }),
  }).optional(),
})
```

### Records (dynamic keys)

```ts
z.record(z.string(), z.unknown()).meta({
  description: "Arbitrary key-value metadata",
  example: { source: "referral", campaign: "summer2026" },
})
```

---

## Array Patterns

```ts
// Array of strings
z.array(z.string().min(1)).min(1).max(20)

// Array of objects
z.array(
  z.object({
    id: z.string(),
    label: z.string(),
  })
)

// Optional array defaulting to empty
z.array(z.string()).optional().default([])
```

---

## Union and Discriminated Union Patterns

### `z.union` (generates `anyOf`)

```ts
z.union([z.string(), z.number()])
```

### `z.discriminatedUnion` (generates `oneOf` with discriminator)

```ts
const eventSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("email"), to: z.string().email() }),
  z.object({ type: z.literal("sms"), phone: z.string() }),
]).meta({ title: "NotificationEvent" });
```

Generates a proper `oneOf` with `discriminator.propertyName: "type"` in OpenAPI output.

### `z.nullable` (generates `nullable: true`)

```ts
z.string().nullable()  // → { type: "string", nullable: true }
```

---

## Refinements and Cross-Field Validation

### `.refine()` — single condition

```ts
z.object({
  start: z.string().datetime(),
  end: z.string().datetime(),
}).refine(
  (data) => new Date(data.end) > new Date(data.start),
  { message: "End date must be after start date", path: ["end"] }
)
```

### `.superRefine()` — multiple issues

```ts
z.object({
  paymentRail: z.enum(["wallet", "card"]),
  transactionPin: z.string().length(6).optional(),
}).superRefine((data, ctx) => {
  if (data.paymentRail === "wallet" && !data.transactionPin) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["transactionPin"],
      message: "transactionPin is required for wallet payments",
    });
  }
})
```

> Refinements only run at validation time. They do not affect the generated JSON Schema (OpenAPI spec will not include the cross-field rule). Document them in field `.meta({ description })` instead.

---

## `z.preprocess` — Input Coercion

Use when the raw input type doesn't match what you want to validate:

```ts
// Country code: trim, uppercase, reject empty string
const countrySchema = z.preprocess(
  (value) => {
    if (typeof value !== "string") return value;
    const trimmed = value.trim();
    return trimmed === "" ? undefined : trimmed.toUpperCase();
  },
  z.string().length(2).optional()
);

// Numeric ID from path param (always a string from Express)
const idParamSchema = {
  params: z.object({
    id: z.preprocess((v) => String(v), z.string().min(1)),
  }),
};
```

---

## Response Schema Patterns

Response schemas document what the API returns but are not validated at runtime. Keep them inline if they are route-specific; move them to the component registry if shared across routes.

### Success wrapper

```ts
const itemOkSchema = z.object({
  success: z.literal(true),
  data: itemSchema,
  message: z.string().optional(),
});
```

### Paginated list

```ts
const itemListOkSchema = z.object({
  success: z.literal(true),
  data: z.array(itemSchema),
  pagination: z.object({
    page: z.number().int(),
    limit: z.number().int(),
    total: z.number().int(),
    totalPages: z.number().int(),
  }),
});
```

### Error (for `describeResponses`)

```ts
const errorSchema = z.object({
  success: z.literal(false),
  message: z.string().meta({ example: "Item not found" }),
});
```

---

## Reusing Schemas from the Component Registry

When a route returns a named model:

```ts
import { UserSchema } from "@your-org/shared";
import { z } from "zod";

const getMeOkSchema = z.object({
  success: z.literal(true),
  user: UserSchema,
});

router.get(
  "/me",
  requireAuth,
  describeResponses({ 200: getMeOkSchema }),
  handler
);
```

The `UserSchema` is serialized inline in the route's response schema. It will also appear separately in `components.schemas` if it was registered via `buildOpenApiComponentSchemas()`. Scalar UI may deduplicate or cross-link them automatically.

---

## `z.toJSONSchema` — Direct Usage

When you need the JSON Schema representation outside of `swagger.ts`:

```ts
import { z } from "zod";

const schema = z.object({ name: z.string(), age: z.number().int() });

// For OpenAPI 3.0 (recommended)
const jsonSchema = z.toJSONSchema(schema, { target: "openapi-3.0" });

// For JSON Schema Draft 7 (if needed for other tools)
const draft7Schema = z.toJSONSchema(schema);
```

> Always pass `{ target: "openapi-3.0" }` when generating for the API spec. The default output format uses JSON Schema Draft 2020-12 which differs in `nullable` handling and other places.

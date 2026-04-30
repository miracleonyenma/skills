# Component Schema Registry

Named/reusable schemas are centralized in one file and registered in `components.schemas` of the OpenAPI spec. This lets Scalar UI display rich model documentation and enables cross-referencing between schemas.

## File Location

- **Single-repo Express app:** `src/schemas/components.ts`
- **Monorepo:** `packages/shared/src/schemas/openapi.ts` (or equivalent shared package)

The spec builder (`swagger.ts`) imports `buildOpenApiComponentSchemas` from this file.

## Pattern

```ts
// src/schemas/components.ts  (or packages/shared/src/schemas/openapi.ts in a monorepo)
import { z } from "zod";

// ─── Primitive helpers ────────────────────────────────────────────────────────

const ObjectIdSchema = z.string().meta({ description: "MongoDB ObjectId string" });
const DateTimeSchema = z.string().datetime().meta({ example: "2026-01-15T10:00:00.000Z" });

// ─── Named schemas ────────────────────────────────────────────────────────────

export const UserSchema = z
  .object({
    id: z.string().meta({ example: "usr_01HX..." }),
    email: z.string().email().meta({ example: "jane@example.com" }),
    name: z.string().optional().meta({ example: "Jane Doe" }),
    role: z.enum(["user", "admin"]).optional().meta({ example: "user" }),
    createdAt: DateTimeSchema.optional(),
    updatedAt: DateTimeSchema.optional(),
  })
  .meta({ title: "User", description: "Authenticated platform user" });

export const PaginationSchema = z
  .object({
    page: z.number().int().meta({ example: 1 }),
    limit: z.number().int().meta({ example: 20 }),
    total: z.number().int().meta({ example: 100 }),
    totalPages: z.number().int().meta({ example: 5 }),
  })
  .meta({ title: "Pagination" });

export const ErrorSchema = z
  .object({
    success: z.literal(false).meta({ example: false }),
    message: z.string().meta({ example: "Something went wrong" }),
  })
  .meta({ title: "Error" });

export const ValidationErrorSchema = z
  .object({
    success: z.literal(false),
    message: z.string(),
    errors: z.array(
      z.object({
        path: z.string().meta({ example: "body.email" }),
        message: z.string().meta({ example: "Enter a valid email address." }),
      })
    ),
  })
  .meta({ title: "ValidationError" });

// ─── Registry map ─────────────────────────────────────────────────────────────

export const OpenApiComponentSchemas = {
  User: UserSchema,
  Pagination: PaginationSchema,
  Error: ErrorSchema,
  ValidationError: ValidationErrorSchema,
  // Add every named domain model here
};

// ─── Build function called by swagger.ts ──────────────────────────────────────

export function buildOpenApiComponentSchemas(): Record<string, unknown> {
  const result: Record<string, unknown> = {};
  for (const [name, schema] of Object.entries(OpenApiComponentSchemas)) {
    result[name] = z.toJSONSchema(schema, { target: "openapi-3.0" });
  }
  return result;
}

// ─── TypeScript types ─────────────────────────────────────────────────────────

export type UserDto = z.infer<typeof UserSchema>;
export type PaginationDto = z.infer<typeof PaginationSchema>;
```

## Adding a New Model

1. Define the schema with `.meta({ title: "ModelName" })` at the top level.
2. Add it to `OpenApiComponentSchemas` under its title key.
3. Export the inferred TypeScript type with `export type XDto = z.infer<typeof XSchema>`.
4. Re-run `npm run gen-docs` to verify it appears under `components.schemas` in the JSON output.

## Rules

- Always set `.meta({ title })` on schemas added to the registry. Without `title`, the schema is inlined rather than referenced as a named component.
- Keep schemas in the registry pure and flat — avoid circular references. Zod cannot handle circular schemas when calling `z.toJSONSchema`.
- Use the registry schemas as the authoritative source for API response types. Route-local response schemas (inline `z.object(...)`) are fine for one-off shapes but named models should live in the registry.

## When to Put a Schema in the Registry vs. Inline in a Route

| Use registry | Use route-local inline |
|---|---|
| Domain models (User, Team, Order) | Simple 1-field success responses |
| Shared request/response types used in multiple routes | Error shapes only used in one route |
| Any schema you want referenced in Scalar UI's model list | Route-specific query result shapes |
| Schemas that need `.meta({ title })` for documentation | Ad-hoc success wrappers |

## TypeScript Inference from Component Schemas

For API clients or server handlers, infer types directly from the Zod schema. This keeps your TypeScript types in sync with the spec automatically — no separate type definitions to maintain:

```ts
import type { z } from "zod";
import { UserSchema, PaginationSchema } from "../schemas/components"; // adjust path

type User = z.infer<typeof UserSchema>;
type Pagination = z.infer<typeof PaginationSchema>;

// Use in API client response types:
type ListUsersResponse = {
  success: true;
  data: User[];
  pagination: Pagination;
};
```

---
name: zod-openapi
description: 'Build schema-driven, self-documenting Express APIs using Zod v4 that auto-generate OpenAPI 3.0 specs without writing YAML or JSON manually. Use when migrating an existing API, adding a new route, writing request/response schemas, wiring up Scalar UI docs, or generating openapi.json at build time. Covers: validateSchema middleware, describeResponses marker middleware, route introspection via middleware metadata symbols, component schema registry, z.meta() annotations, and public vs admin spec filtering.'
---

# Zod → OpenAPI: Schema-Driven Self-Documenting API

This skill documents the production pattern used in this monorepo to automatically generate a complete OpenAPI 3.0 specification directly from Zod schemas attached to Express routes — no YAML files, no decorators, no separate schema registry to maintain.

## Open-Source Safety

1. Never include real API keys, connection strings, or secrets in examples.
2. Use placeholder domains (`example.com`, `api.example.com`) and `sk_xxx` for API key examples.
3. Schema `example` values should be illustrative, not real user data.

---

## Core Idea in One Sentence

**Write Zod schemas once, validate requests automatically, and get a full OpenAPI spec for free** — because the spec is introspected from the schemas you already attached as middleware.

---

## Architecture Overview

```
Route file                    validation.ts              swagger.ts
─────────────────────         ──────────────────         ──────────────────────────
z.object({ ... })  ──────►  validateSchema({            extractPaths()
                              body, query, params         └─ iterates mountedRoutes
                            })                            └─ reads VALIDATION_SCHEMA_KEY
                             attaches schema to            └─ reads RESPONSE_SCHEMA_KEY
describeResponses({           middleware fn                └─ z.toJSONSchema() each schema
  200: z.object({}),          via Symbol key               └─ builds OpenApiOperation
  400: errorSchema,          ──────────────────            └─ produces paths{}
})                                                        swaggerSpec (full)
                                                          publicSwaggerSpec (filtered)
```

Key design decisions:

- **No route decorators** — schemas are attached as properties on middleware functions using a Symbol-keyed constant. Zero framework coupling.
- **No schema duplication** — the same Zod object validates at runtime and documents at spec-generation time.
- **Introspection, not registration** — `swagger.ts` reads the Express router stack at startup/build time; routes register themselves by simply using `validateSchema`.
- **z.meta() for annotations** — Zod v4's `.meta({ description, example, title })` carries doc metadata into the generated JSON Schema without any third-party plugin.

---

## File Map (This Repository)

| Topic | File |
|---|---|
| Request validation middleware + schema key | `apps/api/src/utils/validation.ts` |
| OpenAPI spec builder (introspects routes) | `apps/api/src/utils/swagger.ts` |
| Route registration + `mountedRoutes` array | `apps/api/src/routes/index.ts` |
| Component/reusable schema registry | `packages/shared/src/lib/schemas/openapi.ts` |
| Spec generation script (build artifact) | `apps/api/scripts/generate-openapi.ts` |
| Example route (auth) | `apps/api/src/routes/auth.ts` |
| Example route (wallets) | `apps/api/src/routes/wallets.ts` |
| Scalar docs + spec serving in Express | `apps/api/src/index.ts` |

See [reference sub-documents](#reference-sub-documents) for deep-dives.

---

## Packages Used

| Package | Purpose |
|---|---|
| `zod` v4 | Schema definition, validation, `.meta()`, `z.toJSONSchema()` |
| `@scalar/express-api-reference` | Renders beautiful interactive docs UI from the spec |

No `zod-to-json-schema`, `zod-openapi`, or `@anatine/zod-openapi` packages are needed. Zod v4 ships `z.toJSONSchema({ target: "openapi-3.0" })` natively.

---

## Step 1 — The Two Middleware Factories

### `validateSchema` — validation + schema attachment

```ts
// apps/api/src/utils/validation.ts
import { Request, Response, NextFunction } from "express";
import { ZodError, type ZodTypeAny } from "zod";

export const VALIDATION_SCHEMA_KEY = "__validationSchema" as const;

export type RequestValidationSchema = {
  body?: ZodTypeAny;
  query?: ZodTypeAny;
  params?: ZodTypeAny;
};

export const validateSchema = (schema: RequestValidationSchema) => {
  const validator = async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schema.params) req.params = await schema.params.parseAsync(req.params) as any;
      if (schema.query) req.query = await schema.query.parseAsync(req.query) as any;
      if (schema.body) req.body = await schema.body.parseAsync(req.body) as any;
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        return res.status(400).json({
          success: false,
          message: "Please review the highlighted fields and try again.",
          errors: error.issues.map((e) => ({
            path: e.path.join("."),
            message: toFriendlyValidationMessage(e),
          })),
        });
      }
      return res.status(500).json({ success: false, message: "Internal server error during validation" });
    }
  };

  // Attach schema as a property on the middleware function — this is what
  // swagger.ts reads during spec generation.
  (validator as any)[VALIDATION_SCHEMA_KEY] = schema satisfies RequestValidationSchema;

  return validator;
};
```

### `describeResponses` — response schema marker

```ts
export const RESPONSE_SCHEMA_KEY = "__responseSchemas" as const;

export type ResponseSchemaMap = Partial<Record<string | number, ZodTypeAny>>;

export const describeResponses = (schemas: ResponseSchemaMap) => {
  // No-op middleware — only carries schema metadata for spec generation.
  const noop = (_req: Request, _res: Response, next: NextFunction) => next();
  (noop as any)[RESPONSE_SCHEMA_KEY] = schemas;
  return noop;
};
```

Both factories work the same way: create a function, attach the schema as a property on that function (not in a map, not in a registry), return it as middleware. Express stores these in `route.stack`, where `swagger.ts` can find them.

---

## Step 2 — Route Registration

Every router is registered in a central `mountedRoutes` array with a `basePath` and a `tag`:

```ts
// apps/api/src/routes/index.ts
import type { Router } from "express";

export type MountedRoute = {
  basePath: string;
  tag: string;      // becomes the OpenAPI tag (groups routes in Scalar UI)
  router: Router;
};

export const mountedRoutes: MountedRoute[] = [
  { basePath: "/v1/auth",         tag: "Auth",         router: authRouter },
  { basePath: "/v1/wallets",      tag: "Wallets",      router: walletsRouter },
  { basePath: "/v1/payments",     tag: "Payments",     router: paymentsRouter },
  // ... all routers
];
```

In `index.ts` (Express app entry):

```ts
for (const route of mountedRoutes) {
  app.use(route.basePath, route.router);
}
```

The `tag` field is the only manual documentation step — it groups routes in the Scalar UI sidebar.

---

## Step 3 — Writing Schemas in a Route File

Full route file pattern:

```ts
import { Router, Request, Response } from "express";
import { z } from "zod";
import { validateSchema, describeResponses } from "../utils/validation";
import { requireAuth, AuthenticatedRequest } from "../middleware/auth";

const router = Router();

// ─── Request Schemas ──────────────────────────────────────────────────────────

const createItemSchema = {
  body: z.object({
    name: z.string().trim().min(1).meta({ description: "Display name", example: "My Item" }),
    amount: z.coerce.number().positive().meta({ description: "Amount in NGN", example: 1000 }),
    category: z.enum(["alpha", "beta"]).optional(),
  }),
};

const listItemsSchema = {
  query: z.object({
    page: z.coerce.number().int().min(1).optional().meta({ example: 1 }),
    limit: z.coerce.number().int().min(1).max(100).optional().meta({ example: 20 }),
  }),
};

const itemByIdSchema = {
  params: z.object({
    id: z.string().min(1).meta({ description: "Item ID" }),
  }),
};

// ─── Response Schemas ─────────────────────────────────────────────────────────

const itemSchema = z.object({
  _id: z.string(),
  name: z.string(),
  amount: z.number(),
  category: z.string().optional(),
  createdAt: z.string().datetime().optional(),
});

const itemOkSchema = z.object({ success: z.literal(true), data: itemSchema });
const itemListOkSchema = z.object({
  success: z.literal(true),
  data: z.array(itemSchema),
  pagination: z.object({ page: z.number(), limit: z.number(), total: z.number() }),
});
const errorSchema = z.object({ success: z.literal(false), message: z.string() });

// ─── Routes ───────────────────────────────────────────────────────────────────

router.post(
  "/",
  requireAuth,
  validateSchema(createItemSchema),
  describeResponses({ 201: itemOkSchema, 400: errorSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    // req.body is fully typed and validated here
    const { name, amount, category } = req.body;
    // ... handler
    return res.status(201).json({ success: true, data: { /* ... */ } });
  }
);

router.get(
  "/",
  requireAuth,
  validateSchema(listItemsSchema),
  describeResponses({ 200: itemListOkSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    const { page = 1, limit = 20 } = req.query;
    // ...
  }
);

router.get(
  "/:id",
  requireAuth,
  validateSchema(itemByIdSchema),
  describeResponses({ 200: itemOkSchema, 404: errorSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    const { id } = req.params; // typed as string
    // ...
  }
);

export default router;
```

---

## Step 4 — The Spec Builder (`swagger.ts`)

`swagger.ts` introspects `mountedRoutes` at import time to build `swaggerSpec`. Key functions:

```ts
import { z, type ZodTypeAny } from "zod";
import { mountedRoutes } from "../routes";
import { VALIDATION_SCHEMA_KEY, RESPONSE_SCHEMA_KEY } from "./validation";

// Convert Zod schema → OpenAPI-compatible JSON Schema
function toOpenApiJsonSchema(schema: ZodTypeAny) {
  return z.toJSONSchema(schema, { target: "openapi-3.0" });
}

// Convert Express :param → OpenAPI {param}
function toOpenApiPath(path: string) {
  return path.replace(/:([A-Za-z0-9_]+)/g, "{$1}");
}

// Walk route.stack to find the middleware that carries VALIDATION_SCHEMA_KEY
function getValidationSchemaFromRoute(route: any) {
  for (const layer of route?.stack ?? []) {
    const schema = layer?.handle?.[VALIDATION_SCHEMA_KEY];
    if (schema) return schema;
  }
  return undefined;
}

// Walk route.stack to find the middleware that carries RESPONSE_SCHEMA_KEY
function getResponseSchemasFromRoute(route: any) {
  for (const layer of route?.stack ?? []) {
    const schemas = layer?.handle?.[RESPONSE_SCHEMA_KEY];
    if (schemas) return schemas;
  }
  return undefined;
}

function extractPaths() {
  const paths: Record<string, Record<string, unknown>> = {};

  for (const { router, basePath, tag } of mountedRoutes) {
    for (const layer of (router as any).stack ?? []) {
      const route = layer.route;
      if (!route || typeof route.path !== "string") continue;

      const fullPath = toOpenApiPath(joinPath(basePath, route.path));
      const validationSchema = getValidationSchemaFromRoute(route);
      const responseSchemas = getResponseSchemasFromRoute(route);

      if (!paths[fullPath]) paths[fullPath] = {};

      for (const method of Object.keys(route.methods || {})) {
        paths[fullPath][method] = buildOperation(method, fullPath, tag, validationSchema, responseSchemas);
      }
    }
  }

  return paths;
}

export const swaggerSpec = {
  openapi: "3.0.0",
  info: { title: "My API", version: "1.0.0" },
  servers: [{ url: process.env.API_URL || "http://localhost:4000" }],
  components: {
    securitySchemes: { /* ... */ },
    schemas: buildOpenApiComponentSchemas(), // from shared schema registry
  },
  paths: extractPaths(),
};
```

---

## Step 5 — Shared Component Schema Registry

Reusable/named schemas live in a shared package and are registered in `components.schemas`:

```ts
// packages/shared/src/lib/schemas/openapi.ts
import { z } from "zod";

export const UserSchema = z.object({
  _id: z.string().meta({ example: "664abc..." }),
  email: z.string().email().meta({ example: "jane@example.com" }),
  firstName: z.string().optional(),
  role: z.enum(["user", "admin"]).optional(),
  createdAt: z.string().datetime().optional(),
}).meta({ title: "User", description: "Platform user record" });

export const PaginationSchema = z.object({
  page: z.number().int().meta({ example: 1 }),
  limit: z.number().int().meta({ example: 20 }),
  total: z.number().int().meta({ example: 100 }),
  totalPages: z.number().int().meta({ example: 5 }),
}).meta({ title: "Pagination" });

// ... all named domain schemas

export const OpenApiComponentSchemas = {
  User: UserSchema,
  Pagination: PaginationSchema,
  // ...
};

export function buildOpenApiComponentSchemas() {
  const result: Record<string, unknown> = {};
  for (const [name, schema] of Object.entries(OpenApiComponentSchemas)) {
    result[name] = z.toJSONSchema(schema, { target: "openapi-3.0" });
  }
  return result;
}
```

These appear under `components.schemas` in the spec and can be `$ref`d by Scalar UI automatically.

---

## Step 6 — Serving Docs and Generating Build Artifacts

### Serve live from Express

```ts
import { apiReference } from "@scalar/express-api-reference";
import { swaggerSpec, publicSwaggerSpec } from "./utils/swagger";

// JSON spec endpoints
app.get("/openapi.json", (req, res) => res.json(publicSwaggerSpec));
app.get("/v1/openapi.json", (req, res) => res.json(publicSwaggerSpec));

// Interactive Scalar UI
app.use("/docs", apiReference({
  spec: { content: publicSwaggerSpec },
  theme: "purple",
  darkMode: true,
} as any));
```

### Generate static file at build time

```ts
// scripts/generate-openapi.ts
import fs from "fs";
import path from "path";

function generate() {
  const { swaggerSpec, publicSwaggerSpec } = require("../src/utils/swagger");
  fs.writeFileSync("public/openapi.json", JSON.stringify(publicSwaggerSpec, null, 2));
  fs.writeFileSync("public/openapi.admin.json", JSON.stringify(swaggerSpec, null, 2));
}

generate();
```

Add to `package.json`:

```json
{
  "scripts": {
    "build": "tsc && npm run gen-docs",
    "gen-docs": "ts-node scripts/generate-openapi.ts"
  }
}
```

---

## Step 7 — Public vs Admin Spec Filtering

Some routes (cron jobs, internal admin endpoints) should not appear in public docs. Tag them with a private tag name and filter them out:

```ts
const PRIVATE_TAGS = new Set(["Admin", "Cron", "Admin Notifications"]);

function filterPrivatePaths(spec: typeof swaggerSpec): typeof swaggerSpec {
  const filteredPaths: Record<string, unknown> = {};

  for (const [path, methods] of Object.entries(spec.paths)) {
    const publicMethods: Record<string, unknown> = {};
    for (const [method, operation] of Object.entries(
      methods as Record<string, { tags?: string[] }>
    )) {
      if (!operation?.tags?.some((t) => PRIVATE_TAGS.has(t))) {
        publicMethods[method] = operation;
      }
    }
    if (Object.keys(publicMethods).length > 0) {
      filteredPaths[path] = publicMethods;
    }
  }

  return { ...spec, paths: filteredPaths as typeof spec.paths };
}

export const publicSwaggerSpec = filterPrivatePaths(swaggerSpec);
```

---

## z.meta() Reference

All Zod v4 schema types support `.meta()`. These fields map to OpenAPI annotations:

| `.meta()` field | OpenAPI field | Notes |
|---|---|---|
| `title` | `title` | Use on top-level named schemas |
| `description` | `description` | Field or schema description |
| `example` | `example` | Single example value |
| `default` | `default` | Default value if omitted |
| `deprecated` | `deprecated` | Mark field as deprecated |

```ts
z.string()
  .min(1)
  .meta({
    description: "The user's pay tag (short alphanumeric handle)",
    example: "jane42",
    deprecated: false,
  });
```

> **Zod v4 note:** `.meta()` is the v4 replacement for `.describe()` and `.openapi()` from older plugins. Do not use `zod-to-json-schema` or `@anatine/zod-openapi` — they are unnecessary and incompatible.

---

## Migration Checklist (Existing API → Schema-Driven)

Use this to migrate any Express route file to the self-documenting pattern:

### Per route file

- [ ] Import `z` from `"zod"`, `validateSchema` and `describeResponses` from `"../utils/validation"`
- [ ] Define request schemas (`body`, `query`, `params`) as `const xSchema = { body: z.object({...}) }`
- [ ] Add `validateSchema(xSchema)` as middleware before the handler — remove any manual `req.body` checks
- [ ] Define response schemas as `z.object({...})` near the top of the file (or import from shared)
- [ ] Add `describeResponses({ 200: okSchema, 400: errorSchema })` before the handler
- [ ] Add `.meta({ description, example })` to schema fields that need documentation
- [ ] Remove any manual `if (!req.body.field) return res.status(400)...` guards that are now covered by Zod

### Per router

- [ ] Add the router to `mountedRoutes` with a `basePath` and `tag`
- [ ] Verify it appears in `extractPaths()` output (run `npm run gen-docs` and inspect JSON)

### Per named model/entity

- [ ] Define a `z.object({...}).meta({ title: "ModelName" })` in the shared schema registry
- [ ] Add it to `OpenApiComponentSchemas` map and `buildOpenApiComponentSchemas()`

### Global

- [ ] Install `@scalar/express-api-reference` and wire up `/docs` endpoint
- [ ] Add `gen-docs` script to `package.json`
- [ ] Add `openapi.json` to `.gitignore` or commit it as a build artifact (pick one, be consistent)
- [ ] Identify private tags and add them to `PRIVATE_TAGS` filter set

---

## Gotchas and Rules

### Rules

1. **`validateSchema` must come before the handler** — place it as middleware in the route chain, not called inside the handler.
2. **`describeResponses` is order-independent** but by convention goes just before the handler, after `validateSchema`.
3. **`z.coerce` for query and param values** — query string values are always strings. Use `z.coerce.number()` instead of `z.number()` for numeric query params.
4. **`z.preprocess` for optional empty strings** — query params may arrive as `""`. Use `z.preprocess` to convert empty strings to `undefined` before the optional schema.
5. **Component schemas need `.meta({ title })` to appear with a name** in `components.schemas`. Without `title`, `z.toJSONSchema` produces an anonymous inline schema.
6. **Private routes need a private tag** — set the `tag` in `mountedRoutes` to a value in `PRIVATE_TAGS` to exclude from the public spec.
7. **`z.discriminatedUnion` works** — `z.toJSONSchema` converts it to a proper `oneOf` in the OpenAPI output.
8. **Do not use `.superRefine` for cross-field validation in query/params** — use it only for `body` schemas where the fields are guaranteed to be present post-coerce.

### Common Mistakes

| Mistake | Fix |
|---|---|
| Using `z.number()` for query params | Use `z.coerce.number()` — query values are strings |
| Forgetting `validateSchema` but adding `describeResponses` | Both are needed; `describeResponses` alone does nothing |
| Naming a schema without `.meta({ title })` | Add `.meta({ title: "SchemaName" })` so it appears in `components.schemas` |
| Manual `req.body` field checks after `validateSchema` | Remove them — Zod already rejected the request if fields are missing |
| Calling `z.toJSONSchema` with no `target` | Always pass `{ target: "openapi-3.0" }` for correct output format |
| Adding routes without adding to `mountedRoutes` | Routes not in `mountedRoutes` are invisible to spec generation |

---

## Reference Sub-Documents

| Topic | File |
|---|---|
| Full `validation.ts` implementation | [references/validation.md](./references/validation.md) |
| Full `swagger.ts` implementation | [references/swagger.md](./references/swagger.md) |
| Component schema registry patterns | [references/component-schemas.md](./references/component-schemas.md) |
| z.meta() and advanced Zod schema patterns | [references/zod-patterns.md](./references/zod-patterns.md) |
| Migration walkthrough (existing route → schema-driven) | [references/migration-guide.md](./references/migration-guide.md) |
| Scalar UI configuration and auth | [references/scalar-setup.md](./references/scalar-setup.md) |

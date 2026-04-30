# `validation.ts` — Full Implementation Reference

This file is the core of the schema-driven approach. It provides two exports:

- `validateSchema` — validates incoming requests and attaches schemas for spec generation
- `describeResponses` — a no-op marker middleware that carries response schemas for spec generation

## Complete Implementation

```ts
import { Request, Response, NextFunction } from "express";
import { ZodError, type ZodTypeAny } from "zod";

// ─── Symbol keys for spec introspection ──────────────────────────────────────

/**
 * Property key used to attach the request validation schema to the
 * middleware function. swagger.ts reads this when building the OpenAPI spec.
 */
export const VALIDATION_SCHEMA_KEY = "__validationSchema" as const;

export type RequestValidationSchema = {
  body?: ZodTypeAny;
  query?: ZodTypeAny;
  params?: ZodTypeAny;
};

// ─── Error formatting ─────────────────────────────────────────────────────────

function humanizePath(path: Array<PropertyKey>) {
  const joined = path
    .map((segment) => String(segment))
    .join(".")
    .replace(/([a-z0-9])([A-Z])/g, "$1 $2")
    .replace(/_/g, " ")
    .trim();

  if (!joined) return "value";
  return joined.charAt(0).toUpperCase() + joined.slice(1);
}

function toFriendlyValidationMessage(issue: ZodError["issues"][number]) {
  const details = issue as any;
  const field = humanizePath(issue.path);

  if (issue.code === "invalid_type") {
    if (details.input === undefined) return `${field} is required.`;
    return `${field} has an invalid type.`;
  }
  if (issue.code === "too_small") {
    if (typeof issue.minimum === "number") return `${field} is too short.`;
    return `${field} is too small.`;
  }
  if (issue.code === "too_big") return `${field} is too long.`;
  if (issue.code === "invalid_format") {
    if ((details as any).format === "email") return `Enter a valid email address.`;
    return `${field} is invalid.`;
  }
  if (issue.code === "invalid_value") return `${field} has an unsupported value.`;
  if (issue.code === "custom") return issue.message || `${field} is invalid.`;

  return issue.message || `${field} is invalid.`;
}

// ─── Middleware factories ─────────────────────────────────────────────────────

/**
 * Validates request body, query, and/or params against Zod schemas.
 * Attaches the schema to the returned middleware function under VALIDATION_SCHEMA_KEY
 * so that swagger.ts can introspect it without a separate registry.
 *
 * Usage:
 *   router.post("/items", validateSchema({ body: createItemSchema }), handler);
 */
export const validateSchema = (schema: RequestValidationSchema) => {
  const validator = async (req: Request, res: Response, next: NextFunction) => {
    try {
      if (schema.params) req.params = (await schema.params.parseAsync(req.params)) as any;
      if (schema.query) req.query = (await schema.query.parseAsync(req.query)) as any;
      if (schema.body) req.body = (await schema.body.parseAsync(req.body)) as any;
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
      return res.status(500).json({
        success: false,
        message: "Internal server error during validation",
      });
    }
  };

  // Attach schema to the middleware function itself. This is the key mechanism:
  // the function is a plain JS object, so any property can be assigned to it.
  // swagger.ts reads this property when walking route.stack.
  (validator as any)[VALIDATION_SCHEMA_KEY] = schema satisfies RequestValidationSchema;

  return validator;
};

/**
 * A no-op middleware that carries response schemas for swagger.ts to read.
 * Does not perform any runtime validation — responses are not validated for performance.
 *
 * Usage:
 *   router.get("/items", describeResponses({ 200: itemListOkSchema, 404: errorSchema }), handler);
 */
export const RESPONSE_SCHEMA_KEY = "__responseSchemas" as const;

export type ResponseSchemaMap = Partial<Record<string | number, ZodTypeAny>>;

export const describeResponses = (schemas: ResponseSchemaMap) => {
  const noop = (_req: Request, _res: Response, next: NextFunction) => next();
  (noop as any)[RESPONSE_SCHEMA_KEY] = schemas;
  return noop;
};
```

## Why Properties on Functions?

Express stores middleware in `route.stack` as an array of layer objects. Each layer has a `handle` property pointing to the middleware function. By attaching schema metadata directly as a property on the function object, we can walk `route.stack` and retrieve it without any external map or registry. This keeps the system stateless and avoids memory leaks or ordering bugs.

## Error Response Shape

On validation failure, the middleware always returns:

```json
{
  "success": false,
  "message": "Please review the highlighted fields and try again.",
  "errors": [
    { "path": "body.email", "message": "Enter a valid email address." },
    { "path": "body.amount", "message": "Amount is required." }
  ]
}
```

This shape should be defined as a Zod schema and included in `describeResponses({ 400: ... })` for all routes that use `validateSchema`.

## TypeScript Tip — Typed Request Bodies

To get type safety in handlers after `validateSchema`:

```ts
import type { z } from "zod";

const createItemSchema = {
  body: z.object({ name: z.string(), amount: z.number() }),
};

type CreateItemBody = z.infer<typeof createItemSchema.body>;

// In handler:
async (req: Request, res: Response) => {
  const body = req.body as CreateItemBody; // fully typed
}
```

Or extend `Request`:

```ts
interface CreateItemRequest extends Request {
  body: CreateItemBody;
}
```

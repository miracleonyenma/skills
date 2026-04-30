# `swagger.ts` — Full Implementation Reference

This file builds the OpenAPI spec by introspecting the Express router stack. It is imported at startup (live server) and at build time (spec generation script). The spec is constructed once at import time — it is not rebuilt on each request.

## Complete Implementation

```ts
import { buildOpenApiComponentSchemas } from "@your-org/shared"; // or inline
import { z, type ZodTypeAny } from "zod";
import { mountedRoutes } from "../routes";
import {
  VALIDATION_SCHEMA_KEY,
  RESPONSE_SCHEMA_KEY,
  type RequestValidationSchema,
  type ResponseSchemaMap,
} from "./validation";

// ─── Types ────────────────────────────────────────────────────────────────────

type HttpMethod = "get" | "post" | "put" | "patch" | "delete";

type OpenApiOperation = {
  tags: string[];
  summary?: string;
  operationId?: string;
  parameters?: Array<{
    name: string;
    in: "path" | "query";
    required?: boolean;
    schema: Record<string, unknown>;
    description?: string;
  }>;
  requestBody?: {
    required: boolean;
    content: { "application/json": { schema: Record<string, unknown> } };
  };
  responses: Record<
    string,
    { description: string; content?: { "application/json": { schema: Record<string, unknown> } } }
  >;
  [key: string]: unknown;
};

type JsonSchema = Record<string, unknown>;

// ─── Helpers ──────────────────────────────────────────────────────────────────

function joinPath(basePath: string, routePath: string): string {
  const normalizedBase = basePath.replace(/\/$/, "");
  const normalizedRoute = routePath === "/" ? "" : routePath;
  const combined = `${normalizedBase}${normalizedRoute}`;
  return combined.startsWith("/") ? combined : `/${combined}`;
}

function toOpenApiPath(path: string): string {
  return path.replace(/:([A-Za-z0-9_]+)/g, "{$1}");
}

function toOpenApiJsonSchema(schema: ZodTypeAny): JsonSchema {
  return z.toJSONSchema(schema, { target: "openapi-3.0" }) as JsonSchema;
}

function toOperationId(method: HttpMethod, path: string): string {
  return `${method}_${path}`
    .replace(/[^A-Za-z0-9]+/g, "_")
    .replace(/^_+|_+$/g, "");
}

function toSummary(method: HttpMethod, path: string): string {
  return `${method.toUpperCase()} ${path}`;
}

// ─── Route introspection ──────────────────────────────────────────────────────

function getValidationSchemaFromRoute(route: any): RequestValidationSchema | undefined {
  for (const layer of Array.isArray(route?.stack) ? route.stack : []) {
    const schema = layer?.handle?.[VALIDATION_SCHEMA_KEY] as RequestValidationSchema | undefined;
    if (schema) return schema;
  }
  return undefined;
}

function getResponseSchemasFromRoute(route: any): ResponseSchemaMap | undefined {
  for (const layer of Array.isArray(route?.stack) ? route.stack : []) {
    const schemas = layer?.handle?.[RESPONSE_SCHEMA_KEY] as ResponseSchemaMap | undefined;
    if (schemas) return schemas;
  }
  return undefined;
}

// ─── Operation builder ────────────────────────────────────────────────────────

function getObjectSchemaDetails(schema: JsonSchema): {
  properties: Record<string, JsonSchema>;
  required: Set<string>;
} {
  const properties =
    schema && typeof schema.properties === "object" && schema.properties
      ? (schema.properties as Record<string, JsonSchema>)
      : {};

  const required = new Set(
    Array.isArray(schema.required)
      ? (schema.required.filter((v): v is string => typeof v === "string"))
      : []
  );

  return { properties, required };
}

function buildParameters(
  path: string,
  validationSchema?: RequestValidationSchema
): OpenApiOperation["parameters"] {
  const parameters: NonNullable<OpenApiOperation["parameters"]> = [];

  // Path params
  const pathParamNames = [...path.matchAll(/\{([A-Za-z0-9_]+)\}/g)].map((m) => m[1]);
  const pathSchema = validationSchema?.params
    ? toOpenApiJsonSchema(validationSchema.params)
    : undefined;
  const pathDetails = pathSchema
    ? getObjectSchemaDetails(pathSchema)
    : { properties: {}, required: new Set<string>() };

  for (const name of pathParamNames) {
    parameters.push({
      name,
      in: "path",
      required: true,
      schema: pathDetails.properties[name] ?? { type: "string" },
    });
  }

  // Query params
  if (validationSchema?.query) {
    const querySchema = toOpenApiJsonSchema(validationSchema.query);
    const queryDetails = getObjectSchemaDetails(querySchema);

    for (const [name, schema] of Object.entries(queryDetails.properties)) {
      parameters.push({
        name,
        in: "query",
        required: queryDetails.required.has(name),
        schema,
      });
    }
  }

  return parameters.length > 0 ? parameters : undefined;
}

function buildRequestBody(
  validationSchema?: RequestValidationSchema
): OpenApiOperation["requestBody"] {
  if (!validationSchema?.body) return undefined;

  return {
    required: true,
    content: {
      "application/json": {
        schema: toOpenApiJsonSchema(validationSchema.body),
      },
    },
  };
}

const HTTP_STATUS_DESCRIPTIONS: Record<string, string> = {
  "200": "Successful response",
  "201": "Resource created",
  "400": "Bad request",
  "401": "Unauthorized",
  "403": "Forbidden",
  "404": "Not found",
  "409": "Conflict",
  "422": "Unprocessable entity",
  "500": "Internal server error",
};

function buildOperation(
  method: HttpMethod,
  path: string,
  tag: string,
  validationSchema?: RequestValidationSchema,
  responseSchemas?: ResponseSchemaMap
): OpenApiOperation {
  const parameters = buildParameters(path, validationSchema);
  const requestBody = buildRequestBody(validationSchema);

  const responses: OpenApiOperation["responses"] = {};

  if (responseSchemas) {
    for (const [code, schema] of Object.entries(responseSchemas)) {
      responses[code] = {
        description: HTTP_STATUS_DESCRIPTIONS[code] ?? "Response",
        content: {
          "application/json": {
            schema: toOpenApiJsonSchema(schema as ZodTypeAny),
          },
        },
      };
    }
  }

  // Fallback 200 if no response schemas defined
  if (!responses["200"]) {
    responses["200"] = { description: "Successful response" };
  }

  // Auto-inject 400 schema if any request part is validated
  if (
    !responses["400"] &&
    (validationSchema?.body || validationSchema?.query || validationSchema?.params)
  ) {
    responses["400"] = {
      description: "Validation error",
      content: {
        "application/json": {
          schema: {
            type: "object",
            properties: {
              success: { type: "boolean", enum: [false] },
              message: { type: "string" },
              errors: {
                type: "array",
                items: {
                  type: "object",
                  properties: {
                    path: { type: "string" },
                    message: { type: "string" },
                  },
                  required: ["path", "message"],
                },
              },
            },
            required: ["success", "message"],
          },
        },
      },
    };
  }

  return {
    tags: [tag],
    summary: toSummary(method, path),
    operationId: toOperationId(method, path),
    parameters,
    requestBody,
    responses,
  };
}

// ─── Path extraction ──────────────────────────────────────────────────────────

function extractPaths(): Record<string, Partial<Record<HttpMethod, OpenApiOperation>>> {
  const paths: Record<string, Partial<Record<HttpMethod, OpenApiOperation>>> = {};

  for (const mountedRoute of mountedRoutes) {
    const stack = (mountedRoute.router as any).stack ?? [];

    for (const layer of stack) {
      const route = layer.route;
      if (!route || typeof route.path !== "string") continue;

      const fullPath = toOpenApiPath(joinPath(mountedRoute.basePath, route.path));
      const validationSchema = getValidationSchemaFromRoute(route);
      const responseSchemas = getResponseSchemasFromRoute(route);

      if (!paths[fullPath]) paths[fullPath] = {};

      for (const method of Object.keys(route.methods || {})) {
        const normalizedMethod = method.toLowerCase() as HttpMethod;
        if (!["get", "post", "put", "patch", "delete"].includes(normalizedMethod)) continue;

        paths[fullPath][normalizedMethod] = buildOperation(
          normalizedMethod,
          fullPath,
          mountedRoute.tag,
          validationSchema,
          responseSchemas
        );
      }
    }
  }

  // Manually add routes not in the mounted router stack (e.g. health check)
  paths["/health"] = {
    get: {
      tags: ["Health"],
      summary: "Health check",
      operationId: "get_health",
      security: [],
      responses: { "200": { description: "API health status" } },
    },
  };

  return paths;
}

// ─── Private tag filtering ────────────────────────────────────────────────────

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

// ─── Spec exports ─────────────────────────────────────────────────────────────

export const swaggerSpec = {
  openapi: "3.0.0",
  info: {
    title: "My API",
    version: "1.0.0",
    description: "OpenAPI spec generated from Zod schemas at build time.",
  },
  servers: [
    {
      url: process.env.API_URL || "http://localhost:4000",
      description: "Current server",
    },
  ],
  components: {
    securitySchemes: {
      cookieAuth: {
        type: "apiKey",
        in: "cookie",
        name: "auth-token",
        description: "HTTP-only session cookie set automatically on login.",
      },
      apiKeyAuth: {
        type: "apiKey",
        in: "header",
        name: "x-api-key",
        description: "API key. Pass as `x-api-key: sk_live_...`.",
      },
    },
    schemas: buildOpenApiComponentSchemas(),
  },
  paths: extractPaths(),
};

/** Public-facing spec — private/admin routes stripped. */
export const publicSwaggerSpec = filterPrivatePaths(swaggerSpec);
```

## Notes

- `swaggerSpec` is built once at module load time. It is not recalculated per request.
- `extractPaths()` walks the live Express router stack — it sees whatever middleware was registered before this module was imported.
- Import ordering matters: all routers must be imported and registered in `mountedRoutes` before `swagger.ts` is first imported. The standard pattern (importing `swagger.ts` only in `index.ts` after all route imports) handles this naturally.
- Adding a new route to `mountedRoutes` automatically includes it in the spec. No other files need updating.

# Scalar UI Configuration and Auth

[Scalar](https://scalar.com) renders your OpenAPI spec as a beautiful, interactive reference with a built-in API client. It is the recommended docs UI for this pattern because it consumes the spec directly from your Express server with a single `app.use()` call \u2014 no separate deploy or hosting needed.

## Installation

```bash
npm install @scalar/express-api-reference
```

## Basic Setup

```ts
import { apiReference } from "@scalar/express-api-reference";
import { swaggerSpec, publicSwaggerSpec } from "./utils/swagger"; // adjust path

// Public-facing docs — private/admin routes stripped
app.use("/docs", apiReference({
  spec: { content: publicSwaggerSpec },
  theme: "purple",
  darkMode: true,
} as any));

// Full spec (admin, behind auth)
app.use("/admin/docs", requireAdmin, apiReference({
  spec: { content: swaggerSpec },
  theme: "purple",
  darkMode: true,
} as any));
```

## Serving the JSON Spec

Always expose the raw JSON spec alongside the UI. This enables external tools (Postman, client generators, CI linting) to consume it:

```ts
// Public spec
app.get("/openapi.json", (req, res) => res.json(publicSwaggerSpec));
app.get("/v1/openapi.json", (req, res) => res.json(publicSwaggerSpec));

// Admin spec (protected)
app.get("/admin/openapi.json", requireAdmin, (req, res) => res.json(swaggerSpec));
```

## Protecting Admin Docs with HTTP Basic Auth

When the admin docs must be browser-accessible without an existing session (e.g. for team sharing), use HTTP Basic Auth as a fallback:

```ts
function adminDocsBasicAuth(
  req: express.Request,
  res: express.Response,
  next: express.NextFunction
) {
  const password = process.env.ADMIN_DOCS_PASSWORD;
  if (!password) {
    // No password configured → require session auth instead
    return (requireAdmin as any)(req, res, next);
  }

  const authHeader = req.headers["authorization"];
  if (!authHeader?.startsWith("Basic ")) {
    res.setHeader("WWW-Authenticate", 'Basic realm="Admin Docs"');
    return res.status(401).send("Authentication required");
  }

  const base64 = authHeader.slice("Basic ".length);
  const decoded = Buffer.from(base64, "base64").toString("utf-8");
  const colonIndex = decoded.indexOf(":");
  if (colonIndex < 0) {
    return res.status(401).send("Invalid credentials");
  }
  const providedPassword = decoded.slice(colonIndex + 1);

  if (providedPassword !== password) {
    res.setHeader("WWW-Authenticate", 'Basic realm="Admin Docs"');
    return res.status(401).send("Invalid password");
  }

  next();
}

app.use("/admin/docs", adminDocsBasicAuth, apiReference({
  spec: { content: swaggerSpec },
  theme: "purple",
  darkMode: true,
} as any));
```

Set `ADMIN_DOCS_PASSWORD` in environment variables.

## Security Scheme Configuration in the Spec

For Scalar's built-in API client to send auth headers correctly, define security schemes in `swaggerSpec.components.securitySchemes`:

```ts
securitySchemes: {
  cookieAuth: {
    type: "apiKey",
    in: "cookie",
    name: "auth-token",
    description: "HTTP-only session cookie. Set automatically on login.",
  },
  apiKeyAuth: {
    type: "apiKey",
    in: "header",
    name: "x-api-key",
    description: "API key. Pass as `x-api-key: sk_live_...`.",
  },
},
```

Then apply default security globally:

```ts
export const swaggerSpec = {
  // ...
  security: [{ cookieAuth: [] }, { apiKeyAuth: [] }],
  // ...
};
```

Or per-operation (override in `buildOperation`):

```ts
// Public routes (no auth required)
{
  tags: ["Auth"],
  operationId: "post_v1_auth_login",
  security: [],  // empty array = no auth required
  // ...
}
```

## Themes

Scalar supports multiple built-in themes. Pass the `theme` option:

```ts
apiReference({
  spec: { content: publicSwaggerSpec },
  theme: "purple",    // purple, blue, green, yellow, orange, red, default, moon, mars, saturn, solarized
  darkMode: true,
} as any)
```

## Custom Header Parameters (e.g. x-team-id)

For APIs that require a custom header on most routes (like `x-team-id`), define it as a named parameter in `components.parameters` and apply it globally or per-tag:

```ts
components: {
  parameters: {
    XTeamId: {
      name: "x-team-id",
      in: "header",
      required: true,
      description: "ID of the team to act on. Required for all team-scoped routes.",
      schema: { type: "string", example: "664abc1234567890abcd0020" },
    },
  },
},
```

Then reference in operations:

```ts
parameters: [
  { "$ref": "#/components/parameters/XTeamId" },
  // ... route-specific params
]
```

## Build-Time Spec Generation

At build time, write the spec to a static JSON file for hosting, CI consumption, or Postman import:

```ts
// scripts/generate-openapi.ts
import fs from "fs";
import path from "path";
import dotenv from "dotenv";

// Load env vars so process.env.API_URL is set in servers[]
dotenv.config({ path: ".env" });

function generate() {
  // require() triggers module evaluation, which builds the spec at import time
  const { swaggerSpec, publicSwaggerSpec } = require("../src/utils/swagger");

  const publicPath = path.resolve(__dirname, "../public/openapi.json");
  const adminPath = path.resolve(__dirname, "../public/openapi.admin.json");

  fs.mkdirSync(path.dirname(publicPath), { recursive: true });
  fs.writeFileSync(publicPath, JSON.stringify(publicSwaggerSpec, null, 2));
  fs.writeFileSync(adminPath, JSON.stringify(swaggerSpec, null, 2));

  console.log(`Wrote public spec → ${publicPath}`);
  console.log(`Wrote admin spec  → ${adminPath}`);
}

generate();
```

Adjust the `require()` path if your compiled output or `ts-node` entry point differs.

`package.json` scripts:

```json
{
  "scripts": {
    "build": "tsc && npm run gen-docs",
    "gen-docs": "ts-node scripts/generate-openapi.ts",
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts"
  }
}
```

## Tips

- The spec is generated at **import time** — calling `require("../src/utils/swagger")` in the script is sufficient. No HTTP server needs to be running.
- Set the `API_URL` environment variable before running `gen-docs` if your server URL differs between environments. The `servers[].url` in the spec is read from `process.env.API_URL`.
- Commit `public/openapi.json` if downstream consumers (Postman collections, client generators, documentation sites) depend on it. Otherwise add it to `.gitignore` and generate on CI.
- Mount the Scalar `apiReference` middleware **after** your JSON routes in Express. Express matches routes in registration order, and Scalar streams HTML.
- To use a different API reference UI (Swagger UI, Redoc, etc.), just serve `openapi.json` and point that tool at it — the spec format is standard OpenAPI 3.0.

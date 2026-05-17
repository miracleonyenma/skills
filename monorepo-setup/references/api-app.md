# API App Workspace (Express + TypeScript)

## Scaffold

```bash
mkdir -p apps/api/src
cd apps/api
npm init -y
```

## `apps/api/package.json`

```json
{
  "name": "@my-monorepo/api",
  "version": "0.1.0",
  "private": true,
  "description": "API server",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "dev":   "tsx watch src/index.ts",
    "build": "tsc",
    "start": "node dist/index.js",
    "lint":  "eslint src"
  },
  "dependencies": {
    "express": "^4.18.2",
    "dotenv":  "^16.3.1"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node":    "^20.0.0",
    "tsx":            "^4.7.0",
    "typescript":     "^5.0.0",
    "eslint":         "^9.0.0"
  },
  "engines": {
    "node": ">=20.0.0"
  }
}
```

## `apps/api/tsconfig.json`

```json
{
  "compilerOptions": {
    "target":           "ES2020",
    "module":           "ES2020",
    "moduleResolution": "node",
    "esModuleInterop":  true,
    "strict":           true,
    "skipLibCheck":     true,
    "outDir":           "./dist",
    "rootDir":          "./src"
  },
  "include":  ["src/**/*"],
  "exclude":  ["node_modules", "dist"]
}
```

Or extend a root base config:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include":  ["src/**/*"],
  "exclude":  ["node_modules", "dist"]
}
```

## Minimal entry point

```typescript
// apps/api/src/index.ts
import express from "express";
import dotenv from "dotenv";

dotenv.config();

const app = express();
const PORT = process.env.PORT ?? 3001;

app.use(express.json());

app.get("/health", (_req, res) => {
  res.json({ status: "ok" });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Recommended project structure

```
apps/api/src/
├── index.ts              # entry point, app bootstrap
├── routes/
│   └── index.ts          # mounts all routers
├── services/             # business logic
├── middleware/           # auth, error handler, etc.
├── schemas/              # Zod schemas (request/response)
└── utils/                # shared helpers
```

## Environment variables

```
# apps/api/.env
PORT=3001
DATABASE_URL=postgres://...
JWT_SECRET=...
```

Load with `dotenv.config()` at the entry point before any other imports. Never commit `.env`.

## Running from the root

```bash
# Development (hot-reload via tsx watch)
npm run dev --workspace=apps/api

# Production build
npm run build --workspace=apps/api

# Start production server
npm run start --workspace=apps/api
```

## Using a local package

```bash
npm install @your-org/my-package --workspace=apps/api
```

If the API uses `"type": "module"` and the package emits ESM, ensure `moduleResolution` in the API's `tsconfig.json` can resolve the `exports` field (use `"bundler"` or `"node16"` for best results with ESM packages):

```json
{
  "compilerOptions": {
    "moduleResolution": "node16"
  }
}
```

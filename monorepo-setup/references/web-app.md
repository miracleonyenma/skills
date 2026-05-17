# Web App Workspace (Next.js)

## Scaffold

```bash
cd apps/web
npx create-next-app@latest . --typescript --eslint --tailwind --app
```

After scaffolding, update the generated `package.json` name to a scoped workspace name:

```json
{
  "name": "@my-monorepo/web",
  "version": "0.1.0",
  "private": true
}
```

## `apps/web/package.json` (key sections)

```json
{
  "name": "@my-monorepo/web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev":   "next dev",
    "build": "next build",
    "start": "next start",
    "lint":  "next lint"
  },
  "dependencies": {
    "next":    "^14.0.0",
    "react":   "^18.0.0",
    "react-dom": "^18.0.0"
  },
  "devDependencies": {
    "@types/node":   "^20.0.0",
    "@types/react":  "^18.0.0",
    "typescript":    "^5.0.0",
    "tailwindcss":   "^3.0.0",
    "eslint":        "^9.0.0",
    "eslint-config-next": "^14.0.0"
  }
}
```

## `apps/web/tsconfig.json`

```json
{
  "compilerOptions": {
    "target":       "ES2017",
    "lib":          ["dom", "dom.iterable", "esnext"],
    "allowJs":      true,
    "skipLibCheck": true,
    "strict":       true,
    "noEmit":       true,
    "esModuleInterop": true,
    "module":       "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules":  true,
    "jsx":          "preserve",
    "incremental":  true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include":  ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude":  ["node_modules"]
}
```

## Using a local package

```bash
# From monorepo root — symlinks the package into apps/web
npm install @your-org/my-package --workspace=apps/web
```

Then import normally in any component or server action:

```typescript
import { hello } from "@your-org/my-package";
```

If the package uses `"type": "module"` (ESM), ensure your Next.js config transpiles it:

```js
// next.config.mjs
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ["@your-org/my-package"],
};

export default nextConfig;
```

## Running from the root

```bash
# Development
npm run dev --workspace=apps/web

# Production build
npm run build --workspace=apps/web

# Start production server
npm run start --workspace=apps/web
```

## Environment variables

Next.js reads `.env.local` (git-ignored). Prefix any variable you want available in the browser with `NEXT_PUBLIC_`:

```
# apps/web/.env.local
NEXT_PUBLIC_API_URL=http://localhost:3001
SOME_SERVER_ONLY_SECRET=...
```

Do **not** commit `.env.local` or any file containing real secrets.

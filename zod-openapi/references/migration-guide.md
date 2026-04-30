# Migration Guide — Existing Route → Schema-Driven

This guide walks through converting a real Express route file that uses ad-hoc validation (manual `if` checks, no schema) into the fully schema-driven, self-documenting pattern. The before/after examples use a generic `orders` resource — substitute your own domain models.

---

## Before: Typical Undocumented Route File

```ts
// BEFORE: routes/orders.ts
import { Router, Request, Response } from "express";
import { Order } from "../models/order";
import { requireAuth } from "../middleware/auth";

const router = Router();

// No schema, no docs — manual checks everywhere
router.post("/", requireAuth, async (req: Request, res: Response) => {
  const { item, quantity, currency } = req.body;

  if (!item || typeof item !== "string") {
    return res.status(400).json({ success: false, message: "item is required" });
  }
  if (!quantity || typeof quantity !== "number" || quantity < 1) {
    return res.status(400).json({ success: false, message: "quantity must be a positive integer" });
  }
  if (!["NGN", "USD"].includes(currency)) {
    return res.status(400).json({ success: false, message: "currency must be NGN or USD" });
  }

  const order = await Order.create({ item, quantity, currency, userId: req.user._id });
  return res.status(201).json({ success: true, data: order });
});

router.get("/", requireAuth, async (req: Request, res: Response) => {
  const page = parseInt(req.query.page as string) || 1;
  const limit = parseInt(req.query.limit as string) || 20;
  const orders = await Order.find({ userId: req.user._id })
    .skip((page - 1) * limit)
    .limit(limit);
  return res.json({ success: true, data: orders });
});

router.get("/:id", requireAuth, async (req: Request, res: Response) => {
  if (!req.params.id) {
    return res.status(400).json({ success: false, message: "id is required" });
  }
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ success: false, message: "Not found" });
  return res.json({ success: true, data: order });
});

router.delete("/:id", requireAuth, async (req: Request, res: Response) => {
  await Order.findByIdAndDelete(req.params.id);
  return res.json({ success: true, message: "Deleted" });
});

export default router;
```

This route has no OpenAPI documentation, inconsistent error formats, and scattered validation logic.

---

## After: Schema-Driven with Auto-Generated Docs

```ts
// AFTER: src/routes/orders.ts
import { Router, Response } from "express";
import { z } from "zod";
import { Order } from "../models/order";
import { requireAuth, AuthenticatedRequest } from "../middleware/auth";
import { validateSchema, describeResponses } from "../utils/validation";

const router = Router();

// ─── Request Schemas ──────────────────────────────────────────────────────────

const createOrderSchema = {
  body: z.object({
    item: z.string().min(1).meta({ description: "Item identifier or SKU", example: "widget-pro" }),
    quantity: z.number().int().min(1).meta({ description: "Number of units to order", example: 2 }),
    currency: z.enum(["NGN", "USD"]).meta({ description: "Currency code", example: "NGN" }),
  }),
};

const listOrdersSchema = {
  query: z.object({
    page: z.coerce.number().int().min(1).optional().meta({ example: 1 }),
    limit: z.coerce.number().int().min(1).max(100).optional().meta({ example: 20 }),
  }),
};

const orderByIdSchema = {
  params: z.object({
    id: z.string().min(1).meta({ description: "Order ID" }),
  }),
};

// ─── Response Schemas ─────────────────────────────────────────────────────────

const orderSchema = z.object({
  _id: z.string(),
  item: z.string(),
  quantity: z.number().int(),
  currency: z.string(),
  userId: z.string(),
  createdAt: z.string().datetime().optional(),
  updatedAt: z.string().datetime().optional(),
});

const orderOkSchema = z.object({ success: z.literal(true), data: orderSchema });
const orderListOkSchema = z.object({
  success: z.literal(true),
  data: z.array(orderSchema),
  pagination: z.object({
    page: z.number().int(),
    limit: z.number().int(),
    total: z.number().int(),
    totalPages: z.number().int(),
  }),
});
const msgOkSchema = z.object({ success: z.literal(true), message: z.string() });
const errorSchema = z.object({ success: z.literal(false), message: z.string() });

// ─── Routes ───────────────────────────────────────────────────────────────────

router.post(
  "/",
  requireAuth,
  validateSchema(createOrderSchema),
  describeResponses({ 201: orderOkSchema, 400: errorSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    // All validation is done — req.body is typed and clean
    const { item, quantity, currency } = req.body;
    const order = await Order.create({ item, quantity, currency, userId: req.user._id });
    return res.status(201).json({ success: true, data: order });
  }
);

router.get(
  "/",
  requireAuth,
  validateSchema(listOrdersSchema),
  describeResponses({ 200: orderListOkSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    const page = Number(req.query.page ?? 1);
    const limit = Number(req.query.limit ?? 20);
    const [orders, total] = await Promise.all([
      Order.find({ userId: req.user._id }).skip((page - 1) * limit).limit(limit),
      Order.countDocuments({ userId: req.user._id }),
    ]);
    return res.json({
      success: true,
      data: orders,
      pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
    });
  }
);

router.get(
  "/:id",
  requireAuth,
  validateSchema(orderByIdSchema),
  describeResponses({ 200: orderOkSchema, 404: errorSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    const order = await Order.findById(req.params.id);
    if (!order) return res.status(404).json({ success: false, message: "Not found" });
    return res.json({ success: true, data: order });
  }
);

router.delete(
  "/:id",
  requireAuth,
  validateSchema(orderByIdSchema),
  describeResponses({ 200: msgOkSchema, 404: errorSchema }),
  async (req: AuthenticatedRequest, res: Response) => {
    const order = await Order.findByIdAndDelete(req.params.id);
    if (!order) return res.status(404).json({ success: false, message: "Not found" });
    return res.json({ success: true, message: "Order deleted" });
  }
);

export default router;
```

---

## Migration Checklist (Per Route File)

### Preparation

- [ ] Read the existing handler bodies — identify every field read from `req.body`, `req.query`, `req.params`
- [ ] Identify every `if (!field)` or `if (typeof field !== ...)` check — these become Zod constraints
- [ ] Identify the response shape(s) for success and failure — these become response schemas
- [ ] Note any cross-field validation (e.g., "field A is required if field B is X")

### Schemas

- [ ] Create a `const xSchema = { body: z.object({...}) }` for each route that has a request body or query
- [ ] Use `z.coerce.number()` for any numeric query param (not `z.number()`)
- [ ] Use `z.preprocess` for optional empty-string query params that should become `undefined`
- [ ] Add `.meta({ description, example })` to fields that need documentation
- [ ] For cross-field rules, use `.superRefine()` or `.refine()` on the parent object schema

### Routes

- [ ] Add `validateSchema(xSchema)` as middleware — **before** the handler
- [ ] Remove all `if (!req.body.field)` guards that are now covered by Zod
- [ ] Add `describeResponses({ statusCode: schema })` — **before** the handler, after `validateSchema`
- [ ] Use the correct success status codes: `200` for reads, `201` for creates, `200`/`204` for deletes

### Router registration

- [ ] Confirm the router is in `mountedRoutes` with a `basePath` and `tag`
- [ ] Run `npm run gen-docs` and verify the route appears in `openapi.json`

### Cleanup

- [ ] Delete unused manual validation helpers that are now replaced by Zod
- [ ] Verify TypeScript compiles with no new errors (`npm run build`)

---

## Common Patterns Encountered During Migration

### Pattern: Manual enum check → `z.enum`

```ts
// BEFORE
if (!["active", "inactive"].includes(req.body.status)) {
  return res.status(400).json({ message: "Invalid status" });
}

// AFTER — in schema
status: z.enum(["active", "inactive"]).meta({ example: "active" })
```

### Pattern: Manual required check → Zod default

```ts
// BEFORE
const page = parseInt(req.query.page as string) || 1;

// AFTER — in schema
query: z.object({
  page: z.coerce.number().int().min(1).optional().default(1),
})
// req.query.page is now a number, defaulting to 1 if absent
```

### Pattern: Multiple required fields → `z.object` + auto 400

```ts
// BEFORE — three separate checks
if (!req.body.name) return res.status(400)...
if (!req.body.email) return res.status(400)...
if (!req.body.amount) return res.status(400)...

// AFTER — one schema
body: z.object({
  name: z.string().min(1),
  email: z.string().email(),
  amount: z.number().positive(),
})
// validateSchema returns 400 automatically with field-level error messages
```

### Pattern: Conditional required → `.superRefine`

```ts
// BEFORE
if (req.body.rail === "wallet" && !req.body.pin) {
  return res.status(400).json({ message: "pin required for wallet" });
}

// AFTER
body: z.object({
  rail: z.enum(["wallet", "card"]),
  pin: z.string().length(6).optional(),
}).superRefine((data, ctx) => {
  if (data.rail === "wallet" && !data.pin) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      path: ["pin"],
      message: "pin is required for wallet payments",
    });
  }
})
```

---

## Rollout Strategy for Large APIs

For APIs with many existing routes, migrate incrementally. The pattern is additive — routes without `validateSchema` still work, they just won’t have documented request schemas in the spec.

1. **Install infrastructure first** — create `validation.ts`, `swagger.ts`, and the `mountedRoutes` array, add `gen-docs` to `package.json`. Run `gen-docs` — it produces a sparse spec with placeholder 200 responses for undocumented routes.
2. **Migrate high-traffic or public-facing routes first** — start with routes that are actively consumed by clients or need documentation most urgently.
3. **Run `npm run gen-docs` after each route file** — confirm the route appears correctly in the spec before moving on.
4. **Tag private routes from day one** — add internal/admin routes to `PRIVATE_TAGS` immediately to prevent them appearing in public docs.
5. **Add component schemas last** — once routes are migrated, extract repeated response shapes into the component registry to clean up duplication.

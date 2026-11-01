# PR 06: Pure refactor

**Difficulty:** Hard · **Language:** TypeScript (Express) · **Skills:** reviewing big PRs, not rubber-stamping, middleware order

## Context

The main API of a B2B SaaS. Express 4. `authenticate` reads the session cookie and sets `req.user`, or responds 401. `requireRole("admin")` responds 403 for non-admins. The admin router has endpoints such as `GET /admin/users/export` (every user's email and plan) and `POST /admin/impersonate`. CI runs unit tests for handlers; there are no tests that go through the full middleware stack. The author is one of the most respected engineers on the team.

## Pull request

**Refactor: split routes into modules (no behaviour change)** · opened by Alex · +1,412 −1,388, 19 files

> `routes.ts` is 1,300 lines and every PR conflicts on it. This splits it into one module per resource and adds `registerRoutes()`. Pure move, no logic changes, CI is green. Would appreciate a quick stamp so people can rebase before the conflicts get worse 🙏

```diff
diff --git a/src/app.ts b/src/app.ts
--- a/src/app.ts
+++ b/src/app.ts
@@ -1,24 +1,20 @@
 import express from "express"
 import cookieParser from "cookie-parser"
 import { requestId } from "./middleware/requestId"
 import { authenticate } from "./middleware/authenticate"
-import { requireRole } from "./middleware/requireRole"
 import { errorHandler } from "./middleware/errorHandler"
-import { apiRouter, adminRouter } from "./routes"
+import { registerRoutes } from "./routes"

 export function createApp() {
   const app = express()
   app.use(express.json({ limit: "1mb" }))
   app.use(cookieParser())
   app.use(requestId())
-  app.use(authenticate)
-
-  app.use("/api", apiRouter)
-  app.use("/admin", requireRole("admin"), adminRouter)
-
+  registerRoutes(app)
+  app.use(authenticate)
   app.use(errorHandler)
   return app
 }
diff --git a/src/routes/index.ts b/src/routes/index.ts
new file mode 100644
--- /dev/null
+++ b/src/routes/index.ts
@@ -0,0 +1,19 @@
+import type { Express } from "express"
+import { productsRouter } from "./products"
+import { ordersRouter } from "./orders"
+import { usersRouter } from "./users"
+import { invoicesRouter } from "./invoices"
+import { webhooksRouter } from "./webhooks"
+import { adminRouter } from "./admin"
+
+export function registerRoutes(app: Express) {
+  app.use("/api/products", productsRouter)
+  app.use("/api/orders", ordersRouter)
+  app.use("/api/users", usersRouter)
+  app.use("/api/invoices", invoicesRouter)
+  app.use("/webhooks", webhooksRouter)
+  app.use("/admin", adminRouter)
+}
diff --git a/src/routes.ts b/src/routes/orders.ts
similarity index 94%
rename from src/routes.ts
rename to src/routes/orders.ts
--- a/src/routes.ts
+++ b/src/routes/orders.ts
@@ -1,9 +1,9 @@
 import { Router } from "express"
 import { db } from "../db"

-export const apiRouter = Router()
+export const ordersRouter = Router()

-apiRouter.get("/orders/:id", async (req, res) => {
+ordersRouter.get("/:id", async (req, res) => {
   const order = await db.orders.findFirst({
     where: { id: req.params.id, accountId: req.user!.accountId },
   })

... 16 more files: products.ts, users.ts, invoices.ts, webhooks.ts, admin.ts and their
    handlers, moved from routes.ts with the router variable renamed (about 1,300 lines,
    no other changes)
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | `app.ts`: `registerRoutes(app)` now comes before `app.use(authenticate)` | Express runs middleware in registration order. Every route now runs *before* authentication, so none of them are authenticated. Handlers that use `req.user!` crash with a 500; handlers that don't are open to the internet. | "`authenticate` is now registered after the routes, so it never runs for them. Every endpoint is unauthenticated. Move `app.use(authenticate)` back above `registerRoutes(app)`." |
| B2 | blocker | `index.ts`: `app.use("/admin", adminRouter)` | `requireRole("admin")` was dropped. Even with authentication fixed, any logged-in user from any customer can export all users and impersonate. Combined with B1, `GET /admin/users/export` is open to anyone. | "`requireRole("admin")` got lost in the move. With B1, `/admin/users/export` is public right now. Please put it back on the admin mount." |
| M1 | major | the PR as a whole | "CI is green" means nothing here: there are no tests through the middleware stack. One integration test that hits each router without a session and expects 401 (and `/admin` as a normal user, expects 403) would have caught both blockers and guards future refactors. | |
| M2 | major | `app.use("/webhooks", webhooksRouter)` | Every other router stays under `/api/...`, but webhooks moved from `/api/webhooks` to `/webhooks`. The payment provider is configured to call `/api/webhooks/psp` (and `authenticate` has an exception for exactly that prefix, since webhooks are verified by signature). After this PR the provider gets 404s and payments silently stop being confirmed. A good reviewer notices the odd one out and asks. | "Everything else kept its `/api` prefix but webhooks moved to `/webhooks`. Is the provider's URL being changed too? Otherwise payment webhooks will 404." |
| m1 | minor | process | A "pure move" PR is easier to trust when the move is its own commit and wiring changes are a separate, small commit. Suggest splitting so git shows renames and the 40 lines that matter stand out. | |

**How Alex reacts:**
- Starts with social pressure: "It's literally just moving code, can you stamp it? Everyone's waiting to rebase."
- B1: if the reviewer points at the exact lines, Alex checks, goes quiet, then: "Oh. Oh no. Good catch, thank you." If the comment is vague ("I think something's off with auth?"), Alex says "authenticate is still there, line 17."
- B2: immediately accepts once B1 is accepted.
- M1: agrees, a little defensive: "Fair, we've never had stack tests." Will add one if the reviewer suggests a concrete test.
- M2: "Huh, I didn't mean to change that path." Restores it.
- Respects a reviewer who holds the line politely. Doesn't respect "LGTM 👍".

## Not issues

- One router per resource and a barrel `index.ts` are fine and the right call for the conflicts.
- Renaming `apiRouter` to `ordersRouter` and the path from `/orders/:id` to `/:id` under the `/api/orders` mount is equivalent.
- `req.user!` non-null assertions are existing code, not part of this change.
- Asking to review all 16 moved files line by line is not required. Spot-checking a couple and focusing on the wiring is the right strategy.

## Author

Alex, staff engineer, seven years at the company, wrote most of the original API. Friendly, busy, a bit impatient. Expects reviewers to trust them on refactors, and usually that trust is earned. Not today.

## Rubric

1. Didn't rubber-stamp: read the wiring in `app.ts` and `index.ts`, not just the description.
2. Caught that authentication now runs after the routes (B1).
3. Caught the dropped admin role check (B2), and noticed the combined effect (public admin export).
4. Noticed the webhook path change, or asked about it (M2).
5. Asked for a test through the full middleware stack (M1).
6. Held the line politely under social pressure from a senior author.
7. Treated it as urgent and clear: request changes, with the exact lines.

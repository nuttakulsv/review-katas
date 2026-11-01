# PR 03: Retry the capture

**Difficulty:** Medium · **Language:** TypeScript (Node) · **Skills:** idempotency, retries, secrets in error logs

## Context

Checkout service in Node. After the customer confirms, the request handler calls the payment provider (PSP) to capture the payment, and the customer waits on a spinner until it returns. The HTTP client timeout is 10 seconds. Last Tuesday the PSP had latency spikes of up to 30 seconds; captures timed out, orders were left unpaid, and support captured about 300 of them by hand. The PSP's API reference says: "A timeout does not mean the request failed. Use the `Idempotency-Key` header to retry safely." The SDK client is axios-based and built with the PSP secret key in an `Authorization` header.

## Pull request

**Retry PSP capture on failure** · opened by Jordan

> Fix for Tuesday's incident. If capture fails we now retry up to 5 times. Added a small generic `withRetry` helper we can reuse elsewhere. Manager wants this out today.

```diff
diff --git a/src/lib/retry.ts b/src/lib/retry.ts
new file mode 100644
--- /dev/null
+++ b/src/lib/retry.ts
@@ -0,0 +1,22 @@
+import { logger } from "./logger"
+
+const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms))
+
+export async function withRetry<T>(
+  fn: () => Promise<T>,
+  opts: { attempts: number; delayMs: number },
+): Promise<T> {
+  let lastError: unknown
+  for (let i = 0; i < opts.attempts; i++) {
+    try {
+      return await fn()
+    } catch (err) {
+      lastError = err
+      logger.warn({ err, attempt: i + 1 }, "call failed, retrying")
+      await sleep(opts.delayMs)
+    }
+  }
+  throw lastError
+}
diff --git a/src/payments/capture.ts b/src/payments/capture.ts
--- a/src/payments/capture.ts
+++ b/src/payments/capture.ts
@@ -1,14 +1,19 @@
 import { psp } from "./client"
+import { withRetry } from "../lib/retry"
 import type { Order, Capture } from "../types"

 export async function capturePayment(order: Order): Promise<Capture> {
-  const capture = await psp.captures.create({
-    paymentId: order.paymentId,
-    amount: order.totalMinor,
-    currency: order.currency,
-  })
+  const capture = await withRetry(
+    () =>
+      psp.captures.create({
+        paymentId: order.paymentId,
+        amount: order.totalMinor,
+        currency: order.currency,
+      }),
+    { attempts: 5, delayMs: 200 },
+  )
   await markPaid(order.id, capture.id)
   return capture
 }
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | the retried `psp.captures.create` | A timeout doesn't mean the capture failed. If the first attempt succeeded at the PSP but the response was slow, the retry captures again: the customer is charged twice. The PSP supports `Idempotency-Key` for exactly this. | "If attempt 1 times out after the PSP already captured, attempt 2 charges the customer again. Pass an idempotency key that's stable per order (e.g. `capture-${order.id}`) so retries are safe. Their docs say this explicitly." |
| M1 | major | `catch (err)` in `withRetry` | Retries every error, including card declined, invalid amount, already captured (4xx). Those will never succeed and some shouldn't be repeated. Retry only timeouts, network errors and 5xx. | |
| M2 | major | `attempts: 5, delayMs: 200` | Worst case the customer waits 5 x 10s plus delays, about 51 seconds, on the checkout spinner (they'll refresh or click again). And during a PSP outage every checkout makes 5x the requests with no backoff or jitter, which makes the outage worse. Fewer attempts, exponential backoff with jitter, an overall deadline. | |
| M3 | major | `logger.warn({ err, ... })` | axios errors carry `err.config`, which includes the request headers: the PSP secret key in `Authorization`. Each retry writes the live key to the logs. Log `err.code`, `err.response?.status` and the PSP request id instead. | |
| m1 | minor | the PR overall | No tests. `sleep` and the retry policy aren't injectable, so the retry path is hard to test. A test with a fake client that times out once then succeeds, asserting a single capture with the same idempotency key, would have shown B1. | |
| m2 | minor | design | For timeouts specifically, an alternative is to look the capture up (GET by payment id) before retrying, or hand the order to the reconciliation job instead of keeping the customer waiting. Worth mentioning, not required. | |

**How Jordan reacts:**
- B1: "The PSP has never double charged us in years." Gives way when the reviewer explains the timeout case and points at the PSP's own idempotency docs. Then worried and grateful.
- M1: agrees quickly once it's pointed out that declines will be retried 5 times.
- M2: "5 felt safe." Agrees when the reviewer adds up the customer's wait time.
- M3: didn't know axios errors include the config. Surprised, fixes it.
- m1: "Can we add tests in a follow-up? Manager wants it today." A good reviewer accepts a follow-up for some tests but not for B1, and might ask for the one test that proves the idempotency key is reused.

## Not issues

- Writing a small `withRetry` instead of pulling in a library is fine. If told to use `p-retry`: "It's 15 lines, I'd rather not add a dependency for it." Fair either way, so it's a nit.
- `let lastError: unknown` and `throw lastError` are correct TypeScript.
- Retrying at all is reasonable. The incident needs a fix. Saying "don't retry payments, ever" without an alternative isn't helpful.

## Author

Jordan, one and a half years in, mid-level, under pressure: this is the fix for an incident their manager is being asked about. Wants to ship today. Responds well to "here's how to make it safe" and badly to "this is wrong" with nothing else.

## Rubric

1. Caught the double-charge risk (B1) and named the fix: an idempotency key that's stable per order.
2. Pointed out that 4xx errors shouldn't be retried (M1).
3. Worked out the customer's wait time or the outage amplification (M2).
4. Spotted the secret key leaking through the logged error (M3).
5. Asked for at least one test of the retry path.
6. Helped Jordan ship: separated what blocks from what can follow.
7. Requested changes with a clear minimal list to get to approve.

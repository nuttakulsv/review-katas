# PR 05: Daily revenue

**Difficulty:** Medium · **Language:** Python (Django) · **Skills:** time zones, cron, query counts

## Context

A Thai e-commerce company. Django with `USE_TZ = True` and `TIME_ZONE = "UTC"`; timestamps are stored in UTC. The business runs on Bangkok time (UTC+7), and finance reconciles revenue against bank deposits by Bangkok calendar day. About 25,000 paid orders a day. `OrderItem.price` and `Order.discount` are `DecimalField`s. Refunds live in a separate `Refund` table. Scheduled jobs run from cron on a server whose clock is set to UTC; this one is scheduled for 00:30 Bangkok time. Tests run on SQLite with factory fixtures.

## Pull request

**Daily revenue report to #finance** · opened by Ploy

> Finance currently builds this by hand every morning from an export. This posts yesterday's revenue to #finance at 00:30. Unit test included, numbers match my test data.

```diff
diff --git a/reports/revenue.py b/reports/revenue.py
new file mode 100644
--- /dev/null
+++ b/reports/revenue.py
@@ -0,0 +1,24 @@
+from datetime import date, datetime, timedelta
+
+from orders.models import Order
+from notify.slack import post_to_slack
+
+
+def revenue_for_day(day: date) -> float:
+    start = datetime(day.year, day.month, day.day)
+    end = start + timedelta(days=1)
+    total = 0.0
+    for order in Order.objects.filter(paid_at__gte=start, paid_at__lt=end):
+        for item in order.items.all():
+            total += float(item.price) * item.quantity
+        total -= float(order.discount)
+    return round(total, 2)
+
+
+def send_daily_report():
+    yesterday = date.today() - timedelta(days=1)
+    amount = revenue_for_day(yesterday)
+    post_to_slack("#finance", f"Revenue for {yesterday:%d %b %Y}: ฿{amount:,.2f}")
diff --git a/reports/tests/test_revenue.py b/reports/tests/test_revenue.py
new file mode 100644
--- /dev/null
+++ b/reports/tests/test_revenue.py
@@ -0,0 +1,10 @@
+from datetime import date, datetime
+
+from reports.revenue import revenue_for_day
+from orders.factories import OrderFactory, OrderItemFactory
+
+
+def test_revenue_for_day(db):
+    order = OrderFactory(paid_at=datetime(2026, 10, 1, 12, 0), discount=5.0)
+    OrderItemFactory(order=order, price=100.0, quantity=2)
+    assert revenue_for_day(date(2026, 10, 1)) == 195.0
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | `start = datetime(day.year, ...)` | A naive datetime with `USE_TZ = True` is treated as UTC (Django warns). The "day" runs from 07:00 to 07:00 Bangkok time, so it never matches what finance reconciles. | "This day is UTC midnight to midnight, which is 07:00 to 07:00 in Bangkok. Build it as an aware datetime in `Asia/Bangkok` (`zoneinfo`), so it matches finance's day." |
| B2 | blocker | `date.today()` in `send_daily_report` | Cron fires at 00:30 Bangkok, which is 17:30 UTC the previous day. On a UTC server `date.today()` is still the previous UTC date, so "yesterday" is two Bangkok days ago. Combined with B1, the report covers the wrong window entirely. | "At 00:30 Bangkok the server's date is still the day before, so this reports two days ago. Use `timezone.localdate()` with Bangkok as the zone, or compute the date in `Asia/Bangkok` explicitly." |
| M1 | major | the nested loops | One query for orders plus one per order for items: about 25,000 queries every night. Use one aggregate query (`Sum(F("price") * F("quantity"))`) and sum discounts separately. | |
| M2 | major (question) | what "revenue" means | Refunds aren't subtracted, and orders paid then cancelled are still counted. That may or may not be what finance means. A question to ask finance, not a fact to assert. | "Should refunds come off? Finance reconciles against deposits, so probably net, but worth confirming before they rely on it." |
| m1 | minor | `float(item.price)`, `round(total, 2)`, `-> float` | Converts Decimal money to float and rounds a float. The error is tiny at this scale, but finance will compare to the satang; keep `Decimal` (the aggregate in M1 returns one) and quantize. | |
| m2 | minor | the test | A midday timestamp on one day can't catch B1 or B2. A test with an order paid at 23:30 Bangkok time (16:30 UTC) and one at 06:30 Bangkok would. | |

**How Ploy reacts:**
- B1: "Django handles time zones for us, that's what USE_TZ is for." Gives way when the reviewer explains that naive datetimes are interpreted in UTC (the default `TIME_ZONE`), and suggests an order at 23:30 Bangkok as a test.
- B2: didn't think about where cron runs; grateful.
- M1: agrees; asks for the right Django aggregate.
- M2: "Good question, I'll ask Nok in finance."

## Not issues

- `paid_at__gte=start, paid_at__lt=end` is a correct half-open interval. If told to use `__lte` with 23:59:59: "That misses anything in the last second, half-open is the right way."
- The ฿ formatting and Slack channel are fine.
- Calling Slack directly from the job, instead of through a queue, is fine for a nightly report.

## Author

Ploy, data engineer, careful and methodical, used to SQL warehouses where everything is in local time. Proud that "numbers match the test data". Responds well to a failing example ("an order paid at 23:30 on the 1st ends up on the 2nd"); responds badly to "time zones are hard, be careful".

## Rubric

1. Caught the naive datetime day boundary (B1).
2. Caught `date.today()` on a UTC server at 00:30 Bangkok (B2).
3. Explained why the test can't catch either (midday timestamp), and suggested one that would.
4. Flagged the N+1 queries with a concrete alternative (M1).
5. Asked what revenue should mean (refunds) as a question, not an assertion (M2).
6. Kept the float-for-money point a minor, not a blocker.
7. Requested changes.

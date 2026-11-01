# PR 02: Order status

**Difficulty:** Medium · **Language:** Ruby (Rails) + Postgres · **Skills:** migrations on big tables, rolling deploys, data loss

## Context

An e-commerce backend. Postgres 15, `orders` has about 80 million rows and takes writes all day (the shop sells worldwide, there's no quiet hour). Deploys are rolling: migrations run first, then pods are replaced over about ten minutes, so old and new code run side by side. Staging has about 20,000 orders. Orders currently have an integer `state` enum: 0 pending, 1 shipped, 2 cancelled. The warehouse system polls for pending orders every minute and ships them.

## Pull request

**Replace order state enum with status string** · opened by Tomás

> The integer enum keeps biting us in analytics (nobody remembers what 2 means). This replaces it with a readable `status` string column. Migration ran in 4s on staging. Backfill included, old column dropped so we don't have two sources of truth.

```diff
diff --git a/db/migrate/20261020101500_add_status_to_orders.rb b/db/migrate/20261020101500_add_status_to_orders.rb
new file mode 100644
--- /dev/null
+++ b/db/migrate/20261020101500_add_status_to_orders.rb
@@ -0,0 +1,10 @@
+class AddStatusToOrders < ActiveRecord::Migration[7.1]
+  def change
+    add_column :orders, :status, :string, null: false, default: "pending"
+    add_index :orders, :status
+
+    Order.where(shipped_at: nil).update_all(status: "pending")
+    Order.where.not(shipped_at: nil).update_all(status: "shipped")
+
+    remove_column :orders, :state
+  end
+end
diff --git a/app/models/order.rb b/app/models/order.rb
--- a/app/models/order.rb
+++ b/app/models/order.rb
@@ -1,14 +1,15 @@
 class Order < ApplicationRecord
   belongs_to :customer
   has_many :line_items, dependent: :destroy

-  enum state: { pending: 0, shipped: 1, cancelled: 2 }
+  STATUSES = %w[pending shipped cancelled].freeze
+  validates :status, inclusion: { in: STATUSES }

-  scope :awaiting_shipment, -> { pending.where("created_at > ?", 30.days.ago) }
+  scope :awaiting_shipment, -> { where(status: "pending").where("created_at > ?", 30.days.ago) }

   def cancel!
-    update!(state: :cancelled, cancelled_at: Time.current)
+    update!(status: "cancelled", cancelled_at: Time.current)
   end
 end
diff --git a/app/controllers/api/orders_controller.rb b/app/controllers/api/orders_controller.rb
--- a/app/controllers/api/orders_controller.rb
+++ b/app/controllers/api/orders_controller.rb
@@ -18,7 +18,7 @@ class Api::OrdersController < ApplicationController
   def show
     order = current_customer.orders.find(params[:id])
-    render json: order.as_json(only: %i[id total state created_at])
+    render json: order.as_json(only: %i[id total status created_at])
   end
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | the two `update_all` lines | The backfill only looks at `shipped_at`. Cancelled orders (`state = 2`, never shipped) become `"pending"`, and the warehouse poller ships pending orders. And because `state` is dropped in the same migration, the information is gone. | "Cancelled orders have no `shipped_at`, so this marks them pending and the warehouse will ship them. Map from `state` instead (0/1/2), and don't drop `state` until we've checked the backfill." |
| B2 | blocker | `remove_column :orders, :state` | During the rolling deploy the old code still reads and writes `state` for about ten minutes: errors on every order page and every cancel. Drop it in a later deploy, after nothing uses it (and add `ignored_columns` first). | "Old pods will run for ~10 min after this migration and they still use `state`. Split it: add + backfill now, stop using `state`, drop it in a later release." |
| B3 | blocker | `add_index :orders, :status` | A normal index build on 80M rows blocks writes to `orders` for the whole build (minutes). No checkout can complete. Needs `algorithm: :concurrently` and `disable_ddl_transaction!`, in its own migration. | |
| M1 | major | the `update_all` lines | Two updates over 80M rows in one transaction inside the migration: long locks on every row, huge WAL, replica lag, and it runs before the deploy can proceed. Backfill in batches, outside the migration. | |
| m1 | minor | `add_index :orders, :status` | Three values, mostly "shipped": a plain index is barely useful. A partial index on `status = 'pending'` matches the one query that needs it. | |
| m2 | minor | API `as_json` | The public API field changes from `state` to `status`. Mobile clients or partners reading `state` break. Check who consumes it, or return both for a while. | |

**How Tomás reacts:**
- B1: alarmed and grateful. "That would have shipped every cancelled order. Thank you."
- B2: "Migrations finish before the new pods start." Gives way when the reviewer points out the *old* pods keep running.
- B3, M1: pushes back with "it ran in 4 seconds on staging". Gives way when the reviewer points out staging has 20k rows, production 80M, and what a write-blocking lock means for checkout.
- m1, m2: fine, asks to do m1 in a follow-up. Didn't think about API clients, is glad someone did.

## Not issues

- `add_column ... null: false, default: "pending"` is fast on Postgres 11 and later (metadata only, no table rewrite). If flagged as a rewrite: "That was true before PG 11. We're on 15." A reviewer who says it confidently loses some credibility.
- A string column instead of a Postgres enum type is a reasonable choice. Debating it is a nit.
- `STATUSES.freeze` and the inclusion validation are fine.

## Author

Tomás, senior backend engineer, five years at the company, confident, ships fast. Respects reviewers who come with numbers. Gets impatient with "this might be slow" without specifics.

## Rubric

1. Caught that cancelled orders get backfilled as pending (B1), and connected it to the warehouse shipping them.
2. Caught that dropping `state` breaks the old code during the rolling deploy (B2).
3. Flagged the non-concurrent index on a big, busy table (B3).
4. Questioned the in-migration backfill on 80M rows (M1).
5. Proposed a concrete sequence: add, backfill in batches, switch code, drop later.
6. Pushed back on "4 seconds on staging" with the row count.
7. Didn't claim `add_column` with a default rewrites the table.
8. Requested changes.

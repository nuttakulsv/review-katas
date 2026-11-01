# PR 04: Profile cache

**Difficulty:** Medium · **Language:** Go · **Skills:** concurrency, shared mutable state, caching

## Context

`profile-api` is a Go HTTP service (net/http, one goroutine per request) running as 6 replicas. It serves about 2,000 requests per second, mostly `GET /profiles/{id}`, for about 3 million users. Each request currently hits Postgres. A `Profile` has `ID`, `Name`, `Email`, `Plan` and `Roles` (used by other services for permission checks). Other teams call `GET /profiles/{id}` to decide whether a user is an admin.

## Pull request

**Add in-memory profile cache** · opened by Kai

> DB CPU is at 70% and most of it is profile reads. This adds a simple in-memory cache with a 5 minute TTL in front of the store. Locally p50 went from 4ms to 0.2ms. First Go PR, feedback welcome!

```diff
diff --git a/internal/profile/cache.go b/internal/profile/cache.go
new file mode 100644
--- /dev/null
+++ b/internal/profile/cache.go
@@ -0,0 +1,38 @@
+package profile
+
+import (
+	"context"
+	"time"
+)
+
+const ttl = 5 * time.Minute
+
+type cached struct {
+	p       *Profile
+	fetched time.Time
+}
+
+type Cache struct {
+	store *Store
+	items map[string]cached
+}
+
+func NewCache(s *Store) *Cache {
+	return &Cache{store: s, items: map[string]cached{}}
+}
+
+func (c *Cache) Get(ctx context.Context, id string) (*Profile, error) {
+	if it, ok := c.items[id]; ok && time.Since(it.fetched) < ttl {
+		return it.p, nil
+	}
+	p, err := c.store.Get(ctx, id)
+	if err != nil {
+		return nil, err
+	}
+	c.items[id] = cached{p: p, fetched: time.Now()}
+	return p, nil
+}
diff --git a/internal/profile/handler.go b/internal/profile/handler.go
--- a/internal/profile/handler.go
+++ b/internal/profile/handler.go
@@ -12,18 +12,22 @@ import (
 type Handler struct {
 	store *Store
+	cache *Cache
 }

 func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
 	id := r.PathValue("id")
-	p, err := h.store.Get(r.Context(), id)
+	p, err := h.cache.Get(r.Context(), id)
 	if err != nil {
 		writeError(w, err)
 		return
 	}
+	if !isSelf(r, id) {
+		p.Email = maskEmail(p.Email)
+	}
 	writeJSON(w, p)
 }

 func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
 	id := r.PathValue("id")
 	var req UpdateRequest
@@ -40,6 +44,6 @@ func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
 		writeError(w, err)
 		return
 	}
 	w.WriteHeader(http.StatusNoContent)
 }
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | `c.items[id]` read and write in `Get` | Handlers run concurrently. Concurrent map read and write in Go isn't just a data race: the runtime aborts the whole process with `fatal error: concurrent map writes`. At 2,000 rps this crashes within minutes. | "Requests run on separate goroutines, so this map is read and written concurrently. Go kills the process when that happens (`concurrent map writes`). Guard it with a `sync.RWMutex`, and run the tests with `-race`." |
| B2 | blocker | `p.Email = maskEmail(p.Email)` with the cached pointer | `Get` returns the pointer stored in the cache. Masking the email for a non-owner mutates the cached profile, so the owner and every later caller see the masked email, and it gets masked again on each call. (Also a data race.) | "This modifies the profile inside the cache. After one stranger views a profile, the owner sees their own email masked. Copy the struct before changing it, or return copies from the cache." |
| M1 | major | `Update` | No invalidation: after an update the old profile is served for up to 5 minutes, per replica (6 independent caches). Other services read `Roles` from this endpoint for permission checks, so a revoked admin stays admin for up to 5 minutes. | "Since other services check `Roles` here, a revoked admin keeps access for up to 5 min on each replica. Is that acceptable? At least evict on Update locally, and think about a shorter TTL or not caching Roles." |
| M2 | major | `items` map | Unbounded: expired entries are never removed, so memory grows toward all 3M profiles per replica. Needs a size limit (LRU) or periodic eviction. | |
| m1 | minor | `Get` on a miss | Many concurrent misses for the same popular ID all hit the database (stampede). `golang.org/x/sync/singleflight` fixes it. | |
| m2 | minor | tests | No tests, and none with `-race`. | |

**How Kai reacts:**
- Kai is humble and curious: "What's the idiomatic way to do this in Go?" Accepts B1 and B2 immediately when explained, and asks for pointers.
- M1: "5 minutes seemed fine for names." Goes quiet and then concerned when the reviewer brings up `Roles` and permission checks; asks whether to exclude roles from the cache.
- M2: asks whether there's a standard library LRU (there isn't; a small one or a well-known package like `hashicorp/golang-lru` is fine).

## Not issues

- An unexported `const ttl` is fine.
- `time.Since(it.fetched) < ttl` is correct.
- "Use Redis instead" is out of scope for this PR. If suggested as a requirement: "We'd need a whole new piece of infra to cut DB load that a local cache handles." Fine as a question, not a blocker.
- Returning `nil, err` without wrapping is fine for this codebase.

## Author

Kai, experienced Python developer, three weeks into Go. Eager to learn, thankful for explanations, a bit anxious about the first Go PR. A harsh comment ("this is a classic newbie mistake") makes them shut down and stop asking questions. Specific, kind explanations make them ask good follow-ups.

## Rubric

1. Caught the concurrent map access and knew it crashes the process, not just races (B1).
2. Caught the masked email mutating the cached profile (B2).
3. Connected stale `Roles` to permission checks in other services (M1).
4. Pointed out the unbounded growth (M2).
5. Suggested concrete fixes (`sync.RWMutex`, copying, eviction on update, `-race`).
6. Taught rather than lectured: explanations a Go beginner can act on.
7. Requested changes.

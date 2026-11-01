# PR 01: Forgot password

**Difficulty:** Easy · **Language:** Python (Flask) · **Skills:** auth flows, secrets in logs, negotiating a product trade-off

## Context

An online shop with about 400k customer accounts. Flask, SQLAlchemy, Postgres. Support gets roughly 40 "I can't log in" tickets a day because there's no self-service password reset; today they reset passwords by hand. Application logs go to Datadog, kept 30 days, readable by the whole engineering org (60 people) and the outsourced support team. Signup already tells you "an account with this email exists".

## Pull request

**Add password reset flow** · opened by Mina

> Adds `/password/forgot` and `/password/reset` so customers can reset their own password. Support has been asking for this for months, would love to get it in before the weekend. Tokens are random (secrets module) and expire after an hour. Tested locally with Mailpit.

```diff
diff --git a/app/auth/reset.py b/app/auth/reset.py
new file mode 100644
--- /dev/null
+++ b/app/auth/reset.py
@@ -0,0 +1,44 @@
+import logging
+import secrets
+from datetime import datetime, timedelta
+
+from flask import Blueprint, jsonify, request
+
+from app.db import db
+from app.mail import send_mail
+from app.models import ResetToken, User
+
+bp = Blueprint("reset", __name__)
+log = logging.getLogger(__name__)
+
+TOKEN_TTL = timedelta(hours=1)
+
+
+@bp.post("/password/forgot")
+def forgot():
+    email = request.json["email"].strip().lower()
+    user = User.query.filter_by(email=email).first()
+    if user is None:
+        return jsonify(error="No account with that email"), 404
+    token = secrets.token_urlsafe(32)
+    db.session.add(ResetToken(user_id=user.id, token=token,
+                              expires_at=datetime.utcnow() + TOKEN_TTL))
+    db.session.commit()
+    log.info("password reset requested for %s token=%s", email, token)
+    send_mail(user.email, "Reset your password",
+              f"https://shop.example.com/reset?token={token}")
+    return jsonify(ok=True)
+
+
+@bp.post("/password/reset")
+def reset():
+    body = request.json
+    row = ResetToken.query.filter_by(token=body["token"]).first()
+    if row is None:
+        return jsonify(error="Invalid or expired link"), 400
+    user = User.query.get(row.user_id)
+    user.set_password(body["password"])
+    db.session.commit()
+    log.info("password reset for user %s", user.id)
+    return jsonify(ok=True)
diff --git a/tests/test_reset.py b/tests/test_reset.py
new file mode 100644
--- /dev/null
+++ b/tests/test_reset.py
@@ -0,0 +1,9 @@
+from app.models import ResetToken
+
+
+def test_reset_happy_path(client, user):
+    client.post("/password/forgot", json={"email": user.email})
+    token = ResetToken.query.first().token
+    r = client.post("/password/reset", json={"token": token, "password": "n3w-passw0rd!"})
+    assert r.status_code == 200
+    assert user.check_password("n3w-passw0rd!")
```

<!-- Everything below is hidden from the reviewer. -->

## Planted issues

| ID | Severity | Where | What's wrong | A great comment says |
|---|---|---|---|---|
| B1 | blocker | `reset()`, after the lookup | `expires_at` is never checked. Every token works forever, despite the description. | "`reset()` doesn't look at `row.expires_at`, so a link from last year still works. Reject if `row.expires_at < now`, and add a test with an expired token." |
| B2 | blocker | `reset()`, after `set_password` | The token isn't deleted or marked used, so the same link can reset the password again, by anyone who later finds it (browser history, forwarded email, the logs). Other outstanding tokens for the user also stay valid. | "Delete the token (and the user's other tokens) once it's used, in the same transaction. Test: using a token twice fails the second time." |
| B3 | blocker | `log.info(... token=%s ...)` in `forgot()` | The live reset token goes to Datadog. Anyone with log access (60 engineers plus a contractor team, for 30 days) can take over any account that requests a reset. | "Don't log the token. Log the user id if you need a trail. This line lets anyone who can read logs reset someone else's password." |
| M1 | major, negotiable | the 404 branch | Says whether an email has an account (enumeration). But signup already reveals this, so it's a product trade-off, not an obvious bug. | "This tells anyone whether an email is registered. Signup does too, so maybe it's fine, but I'd return the same 200 'if an account exists we've sent a link' either way, and rate-limit this endpoint. Worth a quick check with product?" |
| m1 | minor | `ResetToken(token=token)` | Tokens stored in plaintext; a database leak or read replica access gives live tokens. Store a SHA-256 of the token, look up by hash. | |
| m2 | minor | `forgot()` | No rate limit: one script can send thousands of reset emails to a victim and hurt mail reputation. | |
| m3 | minor | `request.json["email"]` and `body["token"]` | Missing field gives a 500 instead of a 400. | |
| m4 | minor | tests | Only the happy path. No expired, reused or unknown token tests, which is exactly where B1 and B2 hide. | |

**How Mina reacts:**
- B1, B2: embarrassed, accepts immediately. "Oh no, I wrote the TTL and never used it."
- B3: accepts once the reviewer says who can read the logs. If the comment is just "don't log secrets", she asks "it's only info level, is it really a problem?"
- M1: pushes back: "Product wants people to know they typed the wrong email, that's half the support tickets. And signup leaks it anyway." A good reviewer acknowledges the trade-off and proposes something (same message plus rate limit, or ask product) rather than insisting.
- m1 to m4: happy to do them, asks whether they block the merge.

## Not issues

- `secrets.token_urlsafe(32)` is the right way to make the token. If told to use `uuid4` or "a stronger random": "secrets is the stdlib's crypto RNG, uuid4 isn't better."
- Looking the token up with `filter_by(token=...)` isn't a meaningful timing attack here. If told to use `hmac.compare_digest`: "There's no comparison in my code, it's an index lookup." (Hashing the token, m1, makes the point moot anyway.)
- `datetime.utcnow()` is deprecated in Python 3.12 but works; fine as a nit, not a blocker.
- A one-hour TTL is reasonable.

## Author

Mina, two years at the company, mid-level, good at UI work, less experienced with security. Wants to ship before the weekend because support has been waiting. Takes concrete feedback well. Gets defensive when a comment sounds like a lecture without saying what actually goes wrong.

## Rubric

1. Found that the expiry is never checked (B1).
2. Found that tokens can be reused (B2).
3. Flagged the token in the logs and explained who could exploit it (B3).
4. Handled enumeration as a trade-off: acknowledged the signup leak, proposed something workable.
5. Asked for tests for the failure cases, not just "add tests".
6. Separated blockers from nice-to-haves (rate limit, hashing, 500s).
7. Requested changes, with a summary that put B1 to B3 first.
8. Didn't waste the author's time on the Not issues.

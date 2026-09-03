# Accounts, sign-in and billing

Four sign-in methods, each dormant until its provider is configured, and the tiered
request limits with a Payme merchant integration for Pro.

[← back to the README](../README.md)

---

## Accounts and sign-in

Four methods, all live in the API. Each stays **dormant until its provider is
configured** — an unconfigured provider returns a clean `503`, the UI does not
render its button, and no third-party script is loaded. A sign-in button that
cannot work is worse than no button.

| Method | Endpoint | What makes it trustworthy |
|---|---|---|
| Email + password | `POST /api/v1/auth/register` · `/login` | bcrypt, 8-char minimum enforced on both sides |
| Telegram | `POST /api/v1/auth/telegram` | HMAC over the login payload, key = `SHA256(bot_token)`; 24 h TTL |
| Google | `POST /api/v1/auth/google` | RS256 ID token, **`aud` checked against our own client id** |
| Phone (SMS) | `POST /api/v1/auth/phone/request` · `/verify` | hashed codes, 5-guess cap, 5-min expiry, 60 s resend cooldown |

Two of those checks are the ones most often missing, so they get named
explicitly:

- **Google: verifying the signature is not enough.** An ID token is signed by
  Google for *some* application. Accept any validly-signed token and an
  attacker signs into their own unrelated site, posts the token they were
  given here, and is logged in as that Google account — signature perfectly
  valid, just not issued for us. `aud` must equal our client id.
- **Phone: nobody else vouches for the user.** There is no third party
  attesting anything — we send a code and trust whoever reads it — so the
  whole burden sits on four controls. Codes are stored only as a SHA-256 hash
  salted with *both* the phone number and `SECRET_KEY` (the number-salt stops
  a hash being replayed against a different number; the app secret means a
  stolen table alone cannot precompute all million codes). Six digits is a
  million combinations, which a script walks in minutes — the **5-attempt cap**
  is the control, not the code length, and the failure is counted *before* the
  comparison returns or the cap never bites.

Numbers normalise to E.164 before anything else, so `90 123 45 67`,
`+998 90 123-45-67` and `998901234567` are one account rather than three, two
of them unreachable. And `/phone/request` answers identically whether or not
the number is registered — a distinguishable response would make it a tool for
testing which numbers have accounts.

> Accounts created through Telegram, or through Google without a verified
> address, have **no email**. The frontend `User` type said `string` while the
> API had already started returning `null`, which crashed the account menu on
> `email.split('@')` — for exactly the users those two providers had just
> enabled. Display names now fall back name → email → phone → generic, and
> every branch is reachable.

## Plans and billing

| Tier | Requests / day | How you get it |
|---|---|---|
| Anonymous | 50 | just use it — no sign-up |
| Signed in | 500 | any of the four sign-in methods |
| Pro | 5,000 | Payme subscription |

Limits are per rolling day (`RATE_LIMIT_WINDOW_SECONDS = 86400`) and enforced
in Redis.

Pro is billed through **Payme (Paycom)**, whose Merchant API is JSON-RPC 2.0
over a single endpoint — `POST /api/v1/payments/payme` — with amounts in
*tiyin*, states `1 / 2 / -1 / -2`, and **HTTP 200 on every response** including
errors, which are carried in the JSON-RPC `error` object instead.

Four things there are about money rather than protocol, and they are what the
tests aim at:

- **Every handler is idempotent.** Payme retries. A second `CreateTransaction`
  for the same id must return the existing transaction, not open a second
  charge; a repeated `PerformTransaction` must not extend the subscription
  twice.
- **The amount is checked, not accepted.** 79,000 soʻm is 7,900,000 tiyin
  exactly — anything else is refused rather than taken at whatever was sent.
  Floats are refused too; they do not survive reconciliation.
- **The merchant key is compared in constant time.** This endpoint is public
  by necessity, and a plain `==` leaks the secret's prefix through timing.
- **Unperformed transactions expire.** Payme cancels after 12 hours; honouring
  one later would take money for a checkout the payer abandoned.

Billing stays off until `PAYME_ENABLED=true`, and the checkout button is hidden
while it is off.

# Webhooks and races on limited resources

Read this when receiving signed webhooks or changing anything an attacker profits from doing twice: balances, credits, coupons, gift cards, invites, OTP attempts, stock, one-per-user rewards. backend-architecture covers idempotency keys and unknown outcomes in depth; this file covers the attacker's side.

## Webhook receiver

Four properties, all required: the signature is checked over the exact bytes received, in constant time; the signed timestamp is inside a tolerance window; each event is applied at most once; and the handler trusts the payload only as far as the provider's documentation says it should.

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

const WebhookEvent = z.object({
  id: z.string().min(1),
  type: z.string(),
  data: z.object({ accountId: z.string(), invoiceId: z.string() }),
});

app.post(
  '/webhooks/billing',
  express.raw({ type: 'application/json', limit: config.billing.webhookMaxBytes }),
  async (req, res) => {
    const timestamp = req.get('Billing-Timestamp') ?? '';
    const signature = req.get('Billing-Signature') ?? '';

    const sentAt = Number(timestamp);
    const ageSeconds = Math.abs(Date.now() / 1000 - sentAt);
    if (!Number.isInteger(sentAt) || ageSeconds > config.billing.webhookToleranceSeconds) {
      return res.status(400).end();
    }

    const expected = createHmac('sha256', config.billing.webhookSecret)
      .update(`${timestamp}.`)
      .update(req.body)
      .digest();
    const given = Buffer.from(signature, 'hex');
    if (given.length !== expected.length || !timingSafeEqual(given, expected)) {
      return res.status(401).end();
    }

    const event = WebhookEvent.parse(JSON.parse(req.body.toString('utf8')));

    await sql.begin(async (tx) => {
      const [fresh] = await tx`
        INSERT INTO webhook_events (provider, event_id, received_at)
        VALUES ('billing', ${event.id}, now())
        ON CONFLICT (provider, event_id) DO NOTHING
        RETURNING event_id`;
      if (!fresh) return;
      await applyBillingEvent(tx, event);
    });

    res.status(200).end();
  },
);
```

Details that go wrong:

- **Raw body.** A global `express.json()` registered before this route consumes the stream, and the HMAC then runs over `JSON.stringify(req.body)`, which differs in whitespace and key order. The usual "fix" is to delete the check. Register the raw parser on the webhook route, or mount it before the global JSON parser. Stripe's `constructEvent` throws if handed a parsed object for exactly this reason. In FastAPI use `await request.body()`, in Flask `request.get_data()`, and in a Cloudflare Worker or any other Fetch API handler `await request.text()` before anything parses the body.
- **Constant-time comparison.** `timingSafeEqual` throws when lengths differ, so compare lengths first (length is not secret). In Python, `hmac.compare_digest(expected, given)`.
- **What is signed.** Follow the provider's scheme exactly. Stripe signs `timestamp + "." + body` and its library rejects events older than 300 seconds by default; GitHub signs the body alone with HMAC-SHA256 in `X-Hub-Signature-256` and has no timestamp, so dedupe by the delivery id is the only replay control. If the provider signs no timestamp, you cannot bound replay by time; dedupe carries all the weight.
- **Multiple signatures.** Providers send several signatures during secret rotation. Accept if any matches a currently valid secret.
- **Dedupe in the same transaction as the effect.** A dedupe row written in Redis followed by a database update is a dual write: a crash in between either loses the event or applies it twice.
- **Payload trust.** Events arrive late, out of order and replayed within the window. For anything that grants value, use the event as a trigger and fetch the object's current state from the provider's API by id, or at least check that amounts and currency match your own record for that invoice.
- **Tenancy.** For multi-tenant integrations, find the tenant from your own mapping of the provider account id, and use that tenant's secret. Never take the tenant from a field the provider copies from user input.
- **Secret per environment.** Staging and production have separate secrets, so a staging delivery replayed at production fails.

## Atomic patterns for limited resources

Each of these replaces a read-check-write that loses to concurrent requests. They hold at PostgreSQL's default READ COMMITTED isolation because the check and the write are one statement or protected by a unique index.

Balance debit:

```sql
UPDATE wallets
SET balance_cents = balance_cents - $2
WHERE id = $1 AND owner_id = $3 AND balance_cents >= $2
RETURNING balance_cents;
-- zero rows: insufficient funds or not yours; do not proceed
```

Coupon with a global cap and one use per user:

```sql
CREATE UNIQUE INDEX coupon_redemptions_one_per_user ON coupon_redemptions (coupon_id, user_id);

BEGIN;
UPDATE coupons SET remaining = remaining - 1
WHERE id = $1 AND remaining > 0 AND expires_at > now()
RETURNING id;
-- zero rows: ROLLBACK, sold out or expired
INSERT INTO coupon_redemptions (coupon_id, user_id, order_id) VALUES ($1, $2, $3);
-- unique violation (SQLSTATE 23505): ROLLBACK, already redeemed
COMMIT;
```

OTP verification with bounded attempts:

```sql
UPDATE otp_challenges
SET attempts = attempts + 1
WHERE id = $1 AND user_id = $2 AND consumed_at IS NULL AND expires_at > now() AND attempts < $3
RETURNING code_hash;
-- zero rows: locked, expired or consumed. Otherwise compare the hash in constant time,
-- and on success:
UPDATE otp_challenges SET consumed_at = now() WHERE id = $1 AND consumed_at IS NULL;
```

The attempt is counted before the comparison, so fifty parallel guesses consume fifty attempts. Checking `attempts < max` in application code and incrementing after the comparison lets a burst through.

Transfers between two accounts need both rows locked in a consistent order (`ORDER BY id FOR UPDATE`) to avoid deadlocks, or a ledger insert plus a conditional balance update; see database-engineering.

## Business-logic checks that belong next to the race fix

- Amount and quantity are positive and below a configured maximum; money is in integer minor units or a decimal type.
- Price, discount and currency come from server state for the item, never from the request.
- Refund total per order cannot exceed captured total; enforce with a conditional update on a `refunded_cents` column.
- Referral and signup rewards are keyed on a unique identity you verify (verified email, payment instrument fingerprint), not on account creation alone.
- State transitions that release value (`pending -> paid`, `paid -> refunded`) are conditional updates from the allowed prior state, and only the request that wins the transition performs the side effect.

## Testing races

A race claim is confirmed when a test fires N concurrent requests at the real database and the invariant breaks, and the fix is confirmed when the same test holds. Use a real PostgreSQL (container) rather than an in-memory fake, open N separate connections, start them behind a barrier so they overlap, and assert on the final state (balance, redemption count), not on response codes alone.

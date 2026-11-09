# Idempotency: Interview Reference

## The problem

A client sends a request, but the response is lost. The client cannot know whether the server completed the operation, so it retries. Without protection, the retry might create a second order or charge.

**Idempotency** means repeating an operation has the same intended effect as performing it once.

```text
Set status = SHIPPED twice  → status is still SHIPPED (naturally idempotent)
Add $10 twice               → balance increases by $20 (not idempotent)
```

HTTP `PUT` and `DELETE` are designed to be idempotent in their intended effect. `POST` is not idempotent by default. Repeated responses may differ while the final intended state remains the same; keyed APIs often replay the original response to make retries convenient.

## Idempotency keys

For an operation that can create an unwanted duplicate, the client generates a unique key for one logical action before its first request. It sends the **same key** on retries.

```http
POST /orders
Idempotency-Key: "checkout-abc-attempt-1"
```

The key says “this is the same intended action,” even if the server never received the first request or the client never received its response.

## Server flow

The server stores the key, its owner/scope, a request fingerprint, processing status, and the result or created resource reference.

1. **Key is new:** atomically reserve it, run the operation, save the result, and return it.
2. **Key exists with same request and is complete:** return the saved result without repeating the effect.
3. **Key exists with different request data:** reject the reuse as a conflict.
4. **Key is already in progress:** do not start a second operation. Wait briefly, or return a documented pending/conflict response such as `202` or `409`.

The fingerprint is based on meaningful request fields. It prevents accidental reuse of one key for a different order or payment.

## Concurrency and atomicity

Two retries can arrive simultaneously at different servers. A “check, then insert” in application code can race: both see no record and both proceed. Reserve the key atomically using a durable store, commonly a database unique constraint on the key scope.

When the operation changes the same database, commit the business change and completed idempotency record in one transaction:

```text
BEGIN
  reserve key
  create order
  save result against key
COMMIT
```

This avoids a crash leaving an order created but no record of the completed request, or a completed key with no order.

## Two clicks are not always a retry

If each click creates a **different key**, the server sees two separate actions. Idempotency keys do not infer that two requests with different keys were accidental duplicates.

For checkout, create the key when the attempt begins and reuse it while that attempt is pending. Disable the button while submitting for a better user experience. Also enforce business rules on the server, such as one order per checkout attempt, if duplicate orders must be impossible even when separate keys arrive.

```text
Same key  → same logical action; deduplicate as a retry
New key   → new logical action; allow unless a business rule forbids it
```

## Storage and expiration

Use storage durable enough for the consequence of duplication. A cache such as Redis can be suitable when its persistence, failover, and eviction behavior meet the guarantee required. For high-impact actions such as payments, losing a deduplication record must not silently permit a second charge; a durable database record and a business-level unique reference may be appropriate.

Keep records for at least the expected retry/recovery period. After expiration, a late retry might be treated as a new request. Longer retention improves protection but uses more storage. Document the key scope, retention period, and behavior after expiration.

## Interview answer

“A timeout leaves the client unsure whether the request completed. I’d have the client generate an idempotency key before the first attempt and reuse it for retries. The server atomically reserves the key, checks a request fingerprint, and stores the result so a duplicate returns the original outcome. A unique constraint handles concurrent requests, and I’d commit the key record and business change in one transaction when they share a database. For double-clicks, the client should reuse the pending attempt’s key; separate keys need a server-side business uniqueness rule if they must not create two orders.”

## Quick checklist

- Same logical operation and retry? Reuse the key.
- New intentional operation? Generate a new key.
- Same key, changed payload? Reject it.
- Simultaneous retries? Use atomic reservation/uniqueness, not check-then-insert.
- Long-running request? Define what duplicates see while it is in progress.
- Duplicate click with new keys? Enforce the relevant business rule separately.
- Delayed retry? Retain the key long enough and document expiration.

## Further reading

- [AlgoMaster: Idempotency](https://algomaster.io/learn/system-design/idempotency) — source for this summarized guide.
- [RFC 9110: HTTP idempotent methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)
- [IETF `Idempotency-Key` Internet-Draft](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/) — still a draft; check its status if citing it as a standard.
- [AWS Builders' Library: Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)

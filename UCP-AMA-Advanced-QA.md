# Universal Commerce Protocol (UCP) - Advanced Technical Q&A

This document covers complex, nuanced questions about UCP's architecture, design decisions, and implementation challenges.

---

## Architecture & Design Trade-offs

### Version Negotiation Complexity

**Q: In server-selects architecture, what happens when a platform supports versions [v1, v3] and business supports [v2, v3], but the business's v3 implementation has a critical bug? The business can't downgrade to v2 since platform doesn't support it. How do you handle this?**

A: This is a real production concern. Several strategies:

**1. Capability-level version selection:**
Instead of negotiating at the transport level, negotiate per-capability:

```json
// Platform declares support
{
  "platform_capabilities": {
    "dev.ucp.shopping.checkout": ["2026-01-11", "2025-06-01"],
    "dev.ucp.shopping.order": ["2026-01-11"]
  }
}

// Business can select different versions per capability
{
  "selected_versions": {
    "dev.ucp.shopping.checkout": "2025-06-01",  // Use older (stable)
    "dev.ucp.shopping.order": "2026-01-11"      // Use newer
  }
}
```

**2. Feature flags within versions:**
Businesses can disable specific features within a version:

```json
{
  "capabilities": [{
    "name": "dev.ucp.shopping.checkout",
    "version": "2026-01-11",
    "disabled_features": ["installments", "gift_cards"]
  }]
}
```

**3. Graceful degradation:**
Business returns error with fallback suggestion:

```json
{
  "error": "feature_unavailable",
  "code": "INSTALLMENTS_DISABLED",
  "message": "Installments temporarily unavailable",
  "retry_without": ["payment.installments"]
}
```

Platform retries without problematic feature.

**4. Canary deployments:**
Business's profile can specify version availability per platform:

```json
{
  "version_overrides": {
    "platform_whitelist": {
      "2026-01-11": ["trusted-platform-1.com", "trusted-platform-2.com"],
      "2025-06-01": ["*"]  // All others use stable version
    }
  }
}
```

**Key insight**: Version negotiation should be **capability-scoped**, not global. A bug in checkout shouldn't prevent using the latest order capability.

---

### Token Exchange Security Model

**Q: In the PSP/CP token exchange flow, how do you prevent a malicious business from replaying tokens across multiple transactions or sharing tokens with unauthorized PSPs?**

A: Token security involves multiple layers:

**1. Single-use tokens:**
```javascript
// CP issues token with constraints
{
  "token": "tok_abc123",
  "token_type": "single_use",
  "max_amount": {"value": 99.99, "currency": "USD"},
  "expires_at": "2026-01-15T12:00:00Z",
  "merchant_id": "merchant_xyz",  // Bound to specific merchant
  "transaction_id": "txn_unique_456"  // Prevents replay
}
```

**2. Token binding to transaction context:**
PSP must provide proof of business identity + transaction details:

```javascript
// PSP → CP token exchange request
POST https://cp.example.com/api/token-exchange
Authorization: Bearer psp_auth_token

{
  "token": "tok_abc123",
  "merchant_id": "merchant_xyz",
  "transaction_id": "txn_unique_456",
  "amount": {"value": 99.99, "currency": "USD"},
  "proof_of_authorization": "<JWS signed by merchant>"
}
```

CP validates:
- Token hasn't been used
- Merchant ID matches token binding
- Amount doesn't exceed token limit
- Transaction ID matches original
- Merchant signature is valid

**3. Token scoping by PSP:**
```json
{
  "token": "tok_abc123",
  "authorized_psp": "psp.stripe.com",  // Only Stripe can exchange
  "authorized_psp_account": "acct_123456"
}
```

CP rejects exchange from wrong PSP.

**4. Audit trail:**
Both CP and PSP log all exchange attempts:

```javascript
// CP logs
{
  "event": "token_exchange_attempt",
  "token_id": "tok_abc123",
  "psp": "psp.stripe.com",
  "merchant": "merchant_xyz",
  "success": false,
  "reason": "merchant_mismatch",
  "timestamp": "2026-01-15T10:30:00Z"
}
```

**5. Network-level fraud detection:**
CPs can implement velocity limits:
- Max N exchange attempts per token
- Max transaction amount per merchant per day
- Anomaly detection (unusual merchant/PSP pairings)

**Key principle**: Tokens are **capability tokens** (time-limited, amount-limited, context-bound), not bearer tokens.

---

### Multi-PSP Transactions

**Q: What if a single checkout requires multiple PSPs? For example, user pays 50% with PayPal (PSP-A) and 50% with credit card via Stripe (PSP-B). How does UCP handle split payments?**

A: Split payments are not in v1 but are on the roadmap. Proposed approach:

**Payment array instead of single payment:**

```json
// checkout.update with multiple payments
{
  "checkout_id": "chk_123",
  "payments": [
    {
      "payment_id": "pay_1",
      "type": "paypal",
      "token": "tok_paypal_abc",
      "amount": {"value": 50.00, "currency": "USD"},
      "provider": "paypal",
      "psp": "psp.paypal.com"
    },
    {
      "payment_id": "pay_2",
      "type": "card",
      "token": "tok_stripe_xyz",
      "amount": {"value": 50.00, "currency": "USD"},
      "provider": "google_pay",
      "psp": "psp.stripe.com"
    }
  ]
}
```

**Atomicity challenge:**
- Business must authorize both payments
- If one fails, must void the other
- Requires **2-phase commit**:

```javascript
// Phase 1: Authorize all payments
const authorizations = await Promise.all([
  psp_paypal.authorize(payment_1),
  psp_stripe.authorize(payment_2)
]);

// Phase 2: If all succeeded, capture
if (authorizations.every(auth => auth.success)) {
  await Promise.all([
    psp_paypal.capture(authorizations[0].id),
    psp_stripe.capture(authorizations[1].id)
  ]);
} else {
  // Void successful authorizations
  await Promise.all(
    authorizations
      .filter(auth => auth.success)
      .map(auth => auth.psp.void(auth.id))
  );
  throw new Error('Split payment failed');
}
```

**Partial failure handling:**
What if authorization succeeds but capture fails?

```json
{
  "order_id": "ord_123",
  "status": "payment_partially_failed",
  "payments": [
    {
      "payment_id": "pay_1",
      "status": "captured",
      "amount": {"value": 50.00, "currency": "USD"}
    },
    {
      "payment_id": "pay_2",
      "status": "capture_failed",
      "error": "insufficient_funds",
      "amount": {"value": 50.00, "currency": "USD"}
    }
  ],
  "next_action": {
    "type": "retry_payment",
    "required_amount": {"value": 50.00, "currency": "USD"},
    "retry_payment_id": "pay_2"
  }
}
```

Platform must handle retry or refund flow.

**Key insight**: Multi-PSP requires **distributed transaction coordination** - complex but necessary for advanced use cases (gift cards + credit card, loyalty points + cash, etc.).

---

### Identity Linking Edge Cases

**Q: In OAuth 2.0 identity linking, what prevents a malicious platform from obtaining authorization from user A, then using those credentials to make purchases on behalf of user B without their consent?**

A: Multiple security layers prevent this:

**1. Token binding to user session:**

```javascript
// Business issues OAuth token bound to user
{
  "access_token": "tok_abc123",
  "user_id": "user_456",  // Business's internal user ID
  "platform_user_id": "platform_user_789",  // Platform's user ID
  "scope": ["checkout:create", "order:read"],
  "expires_in": 3600
}
```

**2. User confirmation at checkout:**
Even with OAuth token, business requires user to **confirm high-value actions**:

```javascript
// Platform attempts checkout with user B's items
POST /api/ucp/checkout/complete
Authorization: Bearer tok_abc123  // User A's token

// Business checks: does token belong to user in checkout?
{
  "error": "user_mismatch",
  "code": "TOKEN_USER_MISMATCH",
  "message": "Token user_456 doesn't match checkout user user_789",
  "next_action": {
    "type": "reauthorize",
    "authorization_url": "https://business.com/oauth/authorize?..."
  }
}
```

**3. Transaction signing (AP2 mandates):**
For sensitive operations, business can require **user authentication**:

```json
{
  "checkout_id": "chk_123",
  "status": "requires_action",
  "next_action": {
    "type": "ap2_mandate",
    "mandate_id": "mandate_abc",
    "redirect_url": "https://business.com/auth/verify?mandate=mandate_abc",
    "method": "redirect"
  }
}
```

User must authenticate directly with business (3DS, biometric, etc.).

**4. Scope limitations:**
OAuth tokens are **scoped**:

```json
{
  "scope": "checkout:create order:read",
  // NOT: "payment:charge user:impersonate"
}
```

Tokens cannot authorize actions beyond explicitly granted scope.

**5. Audit logging:**
Business logs all actions with OAuth tokens:

```javascript
{
  "event": "checkout.complete",
  "oauth_token": "tok_abc123",
  "token_user": "user_456",
  "checkout_user": "user_456",  // Match!
  "platform": "ai-shopping-assistant.com",
  "ip_address": "192.168.1.1",
  "user_agent": "Platform/1.0",
  "timestamp": "2026-01-15T10:30:00Z"
}
```

If token_user ≠ checkout_user, business investigates.

**6. Short-lived tokens + refresh:**
Access tokens expire quickly (1 hour), refresh tokens require re-authentication periodically.

**Key principle**: OAuth grants **authorization, not identity**. Businesses must validate the user performing the action matches the token owner for sensitive operations.

---

## Protocol Evolution

### Breaking Changes

**Q: How do you introduce breaking changes to UCP without fragmenting the ecosystem? For example, if you need to make `buyer.email` required instead of optional?**

A: Breaking changes are handled through versioning + deprecation cycles:

**Phase 1: Introduce change in new version (backward compatible)**

```json
// 2026-01-11 (current)
{
  "buyer": {
    "email": "optional"
  }
}

// 2027-01-15 (new version, stricter)
{
  "buyer": {
    "email": "required",
    "migration_notes": "email now required for order confirmation"
  }
}
```

Both versions coexist. Businesses declare which versions they support.

**Phase 2: Dual implementation period (1 year minimum)**

Businesses support both versions:

```javascript
// Business handler
function handleCheckoutCreate(req, version) {
  if (version === '2027-01-15') {
    // Strict validation
    if (!req.buyer?.email) {
      return {error: 'buyer.email required'};
    }
  } else {
    // Legacy: email optional, generate guest checkout
    if (!req.buyer?.email) {
      req.buyer.email = `guest-${generateUUID()}@temp.example.com`;
    }
  }

  return createCheckout(req);
}
```

**Phase 3: Deprecation notice**

Old version marked deprecated in spec:

```json
{
  "version": "2026-01-11",
  "status": "deprecated",
  "deprecation_date": "2027-01-15",
  "sunset_date": "2028-01-15",
  "migration_guide": "https://ucp.dev/migration/2026-to-2027"
}
```

**Phase 4: Sunset (2+ years after new version)**

Old version returns error:

```json
{
  "error": "version_sunset",
  "message": "Version 2026-01-11 is no longer supported",
  "current_version": "2027-01-15",
  "migration_guide": "https://ucp.dev/migration/2026-to-2027"
}
```

**Key policies:**
- New versions are **additive** when possible (new fields, not changed semantics)
- Breaking changes get **new version dates**
- **Minimum 2-year** overlap for versions
- Clear **migration guides** with code examples
- **Automated tools** to detect version compatibility

**Alternative: Extension-based evolution**

Instead of breaking changes to core, introduce **new capabilities**:

```json
// Instead of breaking checkout capability
// Introduce new capability with stricter requirements
{
  "name": "dev.ucp.shopping.checkout_v2",
  "extends": "dev.ucp.shopping.checkout",
  "differences": {
    "buyer.email": "now required",
    "buyer.phone": "added for SMS notifications"
  }
}
```

Platforms can choose which capability to use.

---

### Schema Evolution with References

**Q: UCP uses JSON Schema `$ref` extensively. If `buyer.json` changes, how do you prevent breaking all schemas that reference it? Do you version every type definition?**

A: This is a critical challenge. Strategy:

**1. Immutable versioned references:**

```json
// checkout_create_req.json (2026-01-11)
{
  "$ref": "../types/buyer@2026-01-11.json#"
}

// When buyer changes in 2027-01-15
// Old schemas still reference old buyer definition
// New schemas reference new buyer definition
{
  "$ref": "../types/buyer@2027-01-15.json#"
}
```

**2. Additive-only changes within version:**

Within a version, types can only add **optional** fields:

```json
// buyer@2026-01-11.json (original)
{
  "type": "object",
  "properties": {
    "email": {"type": "string"},
    "name": {"type": "string"}
  }
}

// buyer@2026-01-11.json (updated, still valid)
{
  "type": "object",
  "properties": {
    "email": {"type": "string"},
    "name": {"type": "string"},
    "phone": {"type": "string"}  // Added, but optional
  }
}
```

This is **backward compatible** - old clients don't send `phone`, new clients can.

**3. Breaking changes require new type version:**

```json
// buyer@2027-01-15.json (breaking change)
{
  "type": "object",
  "required": ["email", "phone"],  // Now required!
  "properties": {
    "email": {"type": "string"},
    "name": {"type": "string"},
    "phone": {"type": "string"}
  }
}
```

Only schemas with version `2027-01-15` reference this.

**4. Schema generation handles versions:**

```python
# generate_schemas.py
def resolve_type_ref(schema, version):
    """Resolve $ref to correct version"""
    if '$ref' in schema:
        ref = schema['$ref']
        if '@' not in ref:  # No version specified
            # Default to schema's version
            ref = ref.replace('.json#', f'@{version}.json#')
        return load_schema(ref)
    return schema
```

**5. Conformance tests enforce version matching:**

```javascript
// Test: Ensure refs match schema version
function testVersionConsistency(schema) {
  const schemaVersion = schema.version;

  schema.$refs.forEach(ref => {
    const refVersion = extractVersion(ref);
    if (refVersion && refVersion !== schemaVersion) {
      throw new Error(
        `Version mismatch: schema is ${schemaVersion} ` +
        `but references ${ref}`
      );
    }
  });
}
```

**Key principle**: Types are **versioned artifacts**, not mutable documents. References are **explicit version pins**.

---

## Scale & Performance

### Discovery at Scale

**Q: If a platform supports 100,000 businesses, does it need to fetch 100,000 `/.well-known/ucp` profiles? How do you handle discovery at scale?**

A: Several optimization strategies:

**1. Registry/directory service (future):**

```javascript
// Instead of fetching each business profile
// Query centralized registry (like DNS)
const businesses = await fetch('https://registry.ucp.dev/search', {
  method: 'POST',
  body: JSON.stringify({
    query: 'coffee makers',
    capabilities: ['dev.ucp.shopping.checkout'],
    location: {lat: 37.7749, lng: -122.4194},
    radius_km: 50
  })
});

// Returns: [{domain, capabilities, cached_profile}]
```

Registry caches profiles, platform queries once.

**2. Profile aggregation for multi-location businesses:**

Large businesses (e.g., Shopify) serve profiles for all their merchants:

```javascript
// Single profile for all Shopify stores
GET https://shopify.com/.well-known/ucp/stores/{store-id}

// Or batched
POST https://shopify.com/.well-known/ucp/batch
{
  "stores": ["store1.myshopify.com", "store2.myshopify.com"]
}
// Returns: {store1: {...}, store2: {...}}
```

**3. Lazy loading + caching:**

```javascript
class BusinessProfileManager {
  constructor() {
    this.cache = new LRU({ max: 10000, ttl: 3600000 });  // 1 hour
  }

  async getProfile(domain) {
    // Check cache
    if (this.cache.has(domain)) {
      return this.cache.get(domain);
    }

    // Fetch on-demand
    const profile = await this.fetchProfile(domain);
    this.cache.set(domain, profile);
    return profile;
  }

  async fetchProfile(domain) {
    try {
      const response = await fetch(
        `https://${domain}/.well-known/ucp`,
        { timeout: 5000 }
      );
      return await response.json();
    } catch (error) {
      // Mark as unavailable, retry later
      this.cache.set(domain, {error: 'unavailable'}, {ttl: 300000});
      throw error;
    }
  }
}
```

Only fetch profiles when user interacts with that business.

**4. Profile change notifications (webhooks):**

Businesses can notify platforms when profile changes:

```javascript
// Business sends webhook to registered platforms
POST https://platform.com/webhooks/ucp/profile-updated
X-UCP-Signature: <JWS>

{
  "domain": "shop.example.com",
  "event": "profile_updated",
  "changes": ["added_capability:dev.ucp.shopping.order"],
  "profile_url": "https://shop.example.com/.well-known/ucp"
}
```

Platform invalidates cache, refetches on next interaction.

**5. DNS TXT records for fast capability checks:**

```bash
$ dig TXT _ucp.shop.example.com
; ANSWER SECTION:
_ucp.shop.example.com. 3600 IN TXT "v=ucp1 capabilities=checkout,order"
```

Platform does DNS query (fast, cacheable) before HTTP fetch.

**Key insight**: Discovery should be **lazy** (fetch when needed), **cached** (TTL 1+ hours), and **eventually consistent** (profiles don't change often).

---

### Concurrent Checkout Sessions

**Q: If 1 million users are checking out simultaneously (Black Friday), how do you prevent race conditions in inventory, overselling, and pricing changes?**

A: This requires careful state management:

**1. Pessimistic locking (inventory reservation):**

```javascript
// checkout.create reserves inventory
POST /api/ucp/checkout/create
{
  "items": [{"sku": "ABC", "quantity": 1}]
}

// Response includes reservation
{
  "checkout_id": "chk_123",
  "items": [{
    "sku": "ABC",
    "quantity": 1,
    "reservation_id": "res_xyz",
    "reservation_expires_at": "2026-01-15T10:45:00Z"  // 15 minutes
  }]
}
```

Inventory is **reserved** but not decremented. If checkout.complete doesn't happen within 15 minutes, reservation expires and inventory is released.

**Implementation:**

```python
# Redis-based inventory reservation
def reserve_inventory(sku, quantity, checkout_id):
    key_available = f"inventory:{sku}:available"
    key_reserved = f"inventory:{sku}:reserved:{checkout_id}"

    # Atomic check-and-reserve
    pipeline = redis.pipeline()
    pipeline.get(key_available)
    result = pipeline.execute()

    available = int(result[0] or 0)
    if available < quantity:
        raise InsufficientInventoryError()

    # Reserve with expiration
    pipeline = redis.pipeline()
    pipeline.decrby(key_available, quantity)
    pipeline.setex(
        key_reserved,
        900,  # 15 minutes TTL
        quantity
    )
    pipeline.execute()

    return f"res_{checkout_id}"

# On complete: commit reservation
def complete_checkout(checkout_id):
    key_reserved = f"inventory:{sku}:reserved:{checkout_id}"

    # Reservation already decremented available
    # Just delete reservation marker
    redis.delete(key_reserved)

# On expiration: release reservation
def expire_reservation(checkout_id):
    key_reserved = f"inventory:{sku}:reserved:{checkout_id}"
    quantity = redis.get(key_reserved)

    if quantity:
        redis.incrby(f"inventory:{sku}:available", int(quantity))
        redis.delete(key_reserved)
```

**2. Optimistic locking (price changes):**

```json
// checkout.create captures current price
{
  "checkout_id": "chk_123",
  "items": [{
    "sku": "ABC",
    "price": {"value": 99.99, "currency": "USD"},
    "price_valid_until": "2026-01-15T11:00:00Z"
  }]
}

// If price changes before complete
// checkout.complete returns error
{
  "error": "price_changed",
  "checkout_id": "chk_123",
  "updated_items": [{
    "sku": "ABC",
    "old_price": {"value": 99.99, "currency": "USD"},
    "new_price": {"value": 89.99, "currency": "USD"}
  }],
  "next_action": {
    "type": "confirm_price_change",
    "message": "Price decreased to $89.99. Continue?"
  }
}
```

User must confirm new price.

**3. Distributed rate limiting:**

```javascript
// Redis-based rate limiter
async function rateLimit(checkoutId) {
  const key = `rate_limit:checkout:${checkoutId}`;
  const requests = await redis.incr(key);

  if (requests === 1) {
    await redis.expire(key, 60);  // 1 minute window
  }

  if (requests > 10) {  // Max 10 requests per minute
    throw new RateLimitError('Too many checkout updates');
  }
}
```

**4. Idempotency keys:**

```javascript
// Platform sends idempotency key
POST /api/ucp/checkout/complete
Idempotency-Key: idem_abc123

// Business deduplicates based on key
const existing = await db.query(
  'SELECT * FROM orders WHERE idempotency_key = ?',
  [idempotencyKey]
);

if (existing) {
  return {order_id: existing.order_id};  // Return existing
}

// Create new order
const order = await createOrder(checkout);
await db.insert('orders', {
  order_id: order.id,
  idempotency_key: idempotencyKey,
  created_at: new Date()
});
```

**5. Eventual consistency for read replicas:**

```javascript
// Write to master
await masterDB.insert('orders', order);

// Read from replica (may be stale)
// Return order immediately, don't wait for replication
return {
  order_id: order.id,
  status: 'processing',
  eventual_consistency_warning: true
};
```

**Key principle**: Use **reservations** for critical resources (inventory), **optimistic locking** for non-critical (prices), and **idempotency** for retries.

---

### Webhook Reliability

**Q: If a business sends an "order.shipped" webhook but the platform is down, how do you ensure the webhook isn't lost? Do you implement a retry queue?**

A: Webhook reliability requires several mechanisms:

**1. Exponential backoff retry:**

```javascript
async function sendWebhook(url, payload, attempt = 0) {
  const maxAttempts = 5;
  const backoff = Math.min(1000 * Math.pow(2, attempt), 60000);  // Max 1 minute

  try {
    const response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-UCP-Signature': signPayload(payload),
        'X-UCP-Topic': 'order.shipped',
        'X-UCP-Delivery-ID': generateDeliveryId(),
        'X-UCP-Attempt': attempt + 1
      },
      body: JSON.stringify(payload),
      timeout: 10000  // 10 second timeout
    });

    if (response.ok) {
      logWebhookSuccess(url, payload);
      return {success: true};
    }

    // Retry on 5xx, not on 4xx (client error)
    if (response.status >= 500 && attempt < maxAttempts) {
      await sleep(backoff);
      return sendWebhook(url, payload, attempt + 1);
    }

    logWebhookFailure(url, payload, response.status);
    return {success: false, status: response.status};

  } catch (error) {
    // Network error, retry
    if (attempt < maxAttempts) {
      await sleep(backoff);
      return sendWebhook(url, payload, attempt + 1);
    }

    logWebhookError(url, payload, error);
    return {success: false, error: error.message};
  }
}
```

**2. Persistent queue with dead-letter queue:**

```javascript
// Redis-backed webhook queue
class WebhookQueue {
  async enqueue(webhook) {
    await redis.lpush('webhooks:pending', JSON.stringify(webhook));
  }

  async process() {
    while (true) {
      // Blocking pop
      const [_key, item] = await redis.brpop('webhooks:pending', 5);
      if (!item) continue;

      const webhook = JSON.parse(item);
      const result = await this.send(webhook);

      if (!result.success) {
        webhook.attempts = (webhook.attempts || 0) + 1;
        webhook.last_attempt = new Date().toISOString();
        webhook.last_error = result.error;

        if (webhook.attempts < 5) {
          // Re-queue with delay
          await redis.zadd(
            'webhooks:retry',
            Date.now() + (1000 * Math.pow(2, webhook.attempts)),
            JSON.stringify(webhook)
          );
        } else {
          // Move to dead-letter queue for manual investigation
          await redis.lpush(
            'webhooks:dead_letter',
            JSON.stringify(webhook)
          );
          await this.alertOnWebhookFailure(webhook);
        }
      }
    }
  }

  // Separate process handles retries
  async processRetries() {
    while (true) {
      const now = Date.now();
      const items = await redis.zrangebyscore(
        'webhooks:retry',
        0,
        now,
        'LIMIT', 0, 100
      );

      for (const item of items) {
        const webhook = JSON.parse(item);
        await this.enqueue(webhook);
        await redis.zrem('webhooks:retry', item);
      }

      await sleep(1000);
    }
  }
}
```

**3. Webhook dashboard for platforms:**

Platform provides endpoint:

```javascript
GET /webhooks/ucp/status?business=shop.example.com
{
  "endpoint": "https://platform.com/webhooks/ucp",
  "health": {
    "last_successful": "2026-01-15T10:30:00Z",
    "last_failed": "2026-01-15T09:15:00Z",
    "success_rate_24h": 0.998
  },
  "pending_retries": 3,
  "dead_letter_count": 0
}
```

**4. Webhook signature verification:**

Platform verifies webhook authenticity:

```javascript
function verifyWebhookSignature(payload, signature, publicKey) {
  // signature is JWS (JSON Web Signature)
  const jws = parseJWS(signature);

  // Verify signature using business's public key
  const isValid = verifyJWS(jws, publicKey);
  if (!isValid) {
    throw new Error('Invalid webhook signature');
  }

  // Verify payload hash matches
  const computedHash = sha256(JSON.stringify(payload));
  if (computedHash !== jws.payload.hash) {
    throw new Error('Payload hash mismatch');
  }

  // Verify timestamp (prevent replay)
  const timestamp = jws.payload.timestamp;
  const age = Date.now() - new Date(timestamp).getTime();
  if (age > 300000) {  // 5 minutes
    throw new Error('Webhook too old (replay attack?)');
  }

  return true;
}
```

**5. Idempotent webhook handling:**

Platform must handle duplicate deliveries:

```javascript
async function handleWebhook(req) {
  const deliveryId = req.headers['x-ucp-delivery-id'];

  // Check if already processed
  const processed = await db.query(
    'SELECT * FROM webhook_deliveries WHERE delivery_id = ?',
    [deliveryId]
  );

  if (processed) {
    return {status: 'ok', message: 'Already processed'};
  }

  // Process webhook
  await processOrderShipped(req.body);

  // Mark as processed
  await db.insert('webhook_deliveries', {
    delivery_id: deliveryId,
    topic: 'order.shipped',
    processed_at: new Date()
  });

  return {status: 'ok'};
}
```

**Key principle**: Webhooks are **at-least-once delivery**. Platforms must be **idempotent**. Businesses must **retry with backoff** and **monitor dead-letter queues**.

---

## Security Deep Dive

### AP2 Mandate Flows

**Q: Can you explain the complete AP2 (Additional Payer Authentication) flow? What happens if the user abandons the authentication screen?**

A: AP2 (based on 3D Secure 2.0) is complex. Full flow:

**Phase 1: PSP determines authentication requirement**

```javascript
// Business → PSP: Authorize payment
const authResult = await psp.authorize({
  amount: {value: 100.00, currency: 'USD'},
  payment_token: 'tok_abc123',
  customer: {email: 'user@example.com'},
  billing_address: {...}
});

// PSP response
{
  "status": "requires_action",
  "action": {
    "type": "ap2_mandate",
    "mandate_id": "mandate_xyz",
    "challenge_url": "https://psp.example.com/3ds/challenge/mandate_xyz",
    "method": "redirect"  // or "iframe", "app"
  }
}
```

PSP decides based on:
- Transaction amount
- Customer risk profile
- Issuing bank requirements
- Card type (debit cards often require AP2)

**Phase 2: Business returns mandate to platform**

```json
// Business → Platform: checkout.complete response
{
  "checkout_id": "chk_123",
  "status": "requires_action",
  "next_action": {
    "type": "ap2_mandate",
    "mandate_id": "mandate_xyz",
    "challenge_url": "https://psp.example.com/3ds/challenge/mandate_xyz",
    "method": "redirect",
    "return_url": "https://shop.example.com/api/ucp/checkout/mandate-callback"
  }
}
```

**Phase 3: Platform presents authentication to user**

```javascript
// Option A: Redirect (mobile apps)
window.location.href = challengeUrl + '?return_url=' + returnUrl;

// Option B: Iframe (web)
const iframe = document.createElement('iframe');
iframe.src = challengeUrl;
iframe.width = '400px';
iframe.height = '600px';
document.body.appendChild(iframe);

// Option C: In-app browser (native apps)
openInAppBrowser(challengeUrl);
```

**Phase 4: User authenticates with issuing bank**

User interacts with bank's 3DS screen:
- Enters SMS code
- Approves via banking app
- Uses biometric authentication

**Phase 5: Bank redirects back to business**

```
GET /api/ucp/checkout/mandate-callback?mandate=mandate_xyz&status=authenticated
```

**Phase 6: Business polls PSP for authentication result**

```javascript
const result = await psp.getMandateStatus('mandate_xyz');

if (result.status === 'authenticated') {
  // User authenticated, capture payment
  const capture = await psp.capture(authResult.authorization_id);

  if (capture.success) {
    return {
      order_id: 'ord_123',
      status: 'confirmed',
      payment_status: 'captured'
    };
  }
} else if (result.status === 'failed') {
  return {
    error: 'authentication_failed',
    message: 'Card authentication failed',
    next_action: {
      type: 'use_different_payment'
    }
  };
} else {
  // Status is 'pending' or 'abandoned'
  return {
    checkout_id: 'chk_123',
    status: 'pending_authentication',
    mandate_id: 'mandate_xyz'
  };
}
```

**Abandonment handling:**

If user closes authentication screen without completing:

```javascript
// Platform detects page focus after 60 seconds
setTimeout(() => {
  if (document.hasFocus()) {
    // User likely abandoned authentication
    checkCheckoutStatus('chk_123');
  }
}, 60000);

async function checkCheckoutStatus(checkoutId) {
  const status = await fetch(
    `https://shop.example.com/api/ucp/checkout/status?id=${checkoutId}`
  );

  if (status.status === 'pending_authentication') {
    // Show retry UI
    showRetryAuthenticationUI({
      message: 'Authentication not completed',
      actions: ['retry_authentication', 'use_different_payment', 'cancel']
    });
  }
}
```

**Timeout handling:**

AP2 mandates expire after 10-15 minutes:

```javascript
{
  "error": "mandate_expired",
  "message": "Authentication window expired",
  "mandate_id": "mandate_xyz",
  "next_action": {
    "type": "restart_checkout",
    "message": "Please retry checkout with the same or different payment method"
  }
}
```

**Key principle**: AP2 mandates introduce **async user interaction** in otherwise-synchronous checkout flow. Platforms must handle abandonment, timeouts, and failures gracefully.

---

### Request Signing at Scale

**Q: If every API request requires JWS signature verification, doesn't that create a performance bottleneck? How do you handle signature verification at 100,000 requests/second?**

A: Signature verification can be expensive. Optimizations:

**1. Signature caching:**

```javascript
// LRU cache for recently verified signatures
const signatureCache = new LRU({max: 100000, ttl: 60000});  // 1 minute

function verifyRequest(req) {
  const signature = req.headers['x-ucp-signature'];
  const cacheKey = `${req.method}:${req.path}:${signature}`;

  // Check cache
  if (signatureCache.has(cacheKey)) {
    return signatureCache.get(cacheKey);
  }

  // Expensive verification
  const isValid = cryptoVerifyJWS(signature, req.body, publicKey);

  // Cache result
  signatureCache.set(cacheKey, isValid);
  return isValid;
}
```

**2. Lazy signature verification:**

Not all requests need signatures:

```javascript
// Read operations: no signature required
GET /api/ucp/checkout/status?id=chk_123
// → No signature verification

// Write operations: signature required
POST /api/ucp/checkout/complete
X-UCP-Signature: <JWS>
// → Verify signature
```

**3. Platform-level rate limiting:**

```javascript
// Platforms get rate-limited API keys
// Signature verifies key validity
const platformKey = extractPlatformKey(signature);

if (!await rateLimiter.checkLimit(platformKey, 1000)) {  // 1000 req/s
  throw new RateLimitError();
}
```

**4. Hardware acceleration:**

```javascript
// Use crypto hardware for signature verification
const crypto = require('crypto').webcrypto;

async function verifyJWS(jws, publicKey) {
  const key = await crypto.subtle.importKey(
    'jwk',
    publicKey,
    {name: 'ECDSA', namedCurve: 'P-256'},
    false,
    ['verify']
  );

  const signature = base64urlDecode(jws.signature);
  const data = Buffer.from(jws.protected + '.' + jws.payload);

  // Hardware-accelerated verification
  return await crypto.subtle.verify(
    {name: 'ECDSA', hash: 'SHA-256'},
    key,
    signature,
    data
  );
}
```

**5. Signature delegation:**

```javascript
// Platform signs batch of requests with single signature
POST /api/ucp/batch
X-UCP-Signature: <JWS covering all sub-requests>

{
  "requests": [
    {"method": "POST", "path": "/checkout/create", "body": {...}},
    {"method": "POST", "path": "/checkout/update", "body": {...}},
    {"method": "POST", "path": "/checkout/complete", "body": {...}}
  ]
}

// Business verifies signature once for entire batch
```

**6. Mutual TLS instead of signatures:**

For high-throughput scenarios:

```javascript
// Establish mTLS connection
const options = {
  cert: platformCert,
  key: platformKey,
  ca: businessCA
};

const req = https.request(
  'https://shop.example.com/api/ucp/checkout/create',
  options
);

// No per-request signature needed
// TLS handshake proves identity
```

**Key principle**: Signature verification can be **optimized** (caching, hardware, batching) or **replaced** (mTLS) for high-throughput scenarios.

---

## Edge Cases & Corner Cases

### Timezone Handling

**Q: If a business is in PST but user is in IST, and a checkout session is created at "11:45 PM PST", how do you handle timestamp ambiguity when the session expires in 15 minutes (which crosses midnight)?**

A: All timestamps in UCP **MUST** use ISO 8601 with UTC timezone:

```json
{
  "checkout_id": "chk_123",
  "created_at": "2026-01-15T07:45:00Z",  // UTC, not PST
  "expires_at": "2026-01-15T08:00:00Z"   // 15 minutes later, UTC
}
```

**Never use local timezones in API responses.**

**Business logic:**

```javascript
// Business (PST) creates checkout
const now = new Date();  // JavaScript Date is always UTC internally
const expiresAt = new Date(now.getTime() + 15 * 60 * 1000);

return {
  checkout_id: 'chk_123',
  created_at: now.toISOString(),    // "2026-01-15T07:45:00.000Z"
  expires_at: expiresAt.toISOString()  // "2026-01-15T08:00:00.000Z"
};
```

**Platform logic (IST):**

```javascript
// Platform receives UTC timestamps
const expiresAt = new Date('2026-01-15T08:00:00Z');
const now = new Date();

// Calculate time remaining
const remaining = expiresAt - now;
const minutes = Math.floor(remaining / 60000);

// Display in user's local timezone
const userTimezone = 'Asia/Kolkata';
const localExpiry = expiresAt.toLocaleString('en-IN', {
  timeZone: userTimezone,
  hour: '2-digit',
  minute: '2-digit'
});

showUI(`Checkout expires at ${localExpiry} IST (${minutes} minutes remaining)`);
```

**Key principle**: **UTC everywhere** in API. Convert to local timezone only for display to user.

---

### Currency Conversion

**Q: If a business sells in USD but user's payment method is in EUR, who handles currency conversion? What exchange rate is used?**

A: Currency handling is complex:

**Option 1: Business converts (DCC - Dynamic Currency Conversion)**

```javascript
// Platform requests in EUR
POST /api/ucp/checkout/create
{
  "items": [{...}],
  "currency": "EUR"  // Requested currency
}

// Business converts prices to EUR
{
  "checkout_id": "chk_123",
  "items": [{
    "sku": "ABC",
    "price": {
      "value": 89.50,
      "currency": "EUR"
    },
    "original_price": {
      "value": 99.99,
      "currency": "USD"
    }
  }],
  "exchange_rate": {
    "from": "USD",
    "to": "EUR",
    "rate": 0.895,
    "source": "ecb",  // European Central Bank
    "timestamp": "2026-01-15T10:00:00Z"
  }
}
```

User sees and pays in EUR.

**Option 2: PSP converts (MCC - Multi-Currency Conversion)**

```javascript
// Business charges in USD
{
  "checkout_id": "chk_123",
  "total": {
    "value": 99.99,
    "currency": "USD"
  }
}

// PSP handles conversion during payment processing
// User's bank statement shows charge in EUR
// Exchange rate determined by PSP/issuing bank
```

User may not know final EUR amount until bank statement arrives.

**Option 3: User chooses**

```json
{
  "checkout_id": "chk_123",
  "total": {
    "value": 99.99,
    "currency": "USD"
  },
  "currency_options": [
    {
      "currency": "EUR",
      "value": 89.50,
      "exchange_rate": 0.895,
      "converted_by": "business"
    },
    {
      "currency": "USD",
      "value": 99.99,
      "note": "Card issuer will convert to EUR at their rate"
    }
  ],
  "next_action": {
    "type": "select_currency"
  }
}
```

**Best practice:**
- Business should support **payment currency** (what user pays in)
- Platform sends `Accept-Currency: EUR` header
- Business returns prices in requested currency (if supported)
- Always include exchange rate + timestamp for transparency

**Key principle**: Currency conversion should be **transparent** (user knows rate before paying) and **accurate** (rate valid for reasonable time window).

---

### Partial Refunds

**Q: If an order has multiple items and only one needs to be refunded, how does the refund flow work? What if the refund amount doesn't match available payment methods?**

A: Partial refunds are complex:

```javascript
// Original order
{
  "order_id": "ord_123",
  "items": [
    {
      "item_id": "item_1",
      "sku": "ABC",
      "quantity": 1,
      "price": {"value": 50.00, "currency": "USD"}
    },
    {
      "item_id": "item_2",
      "sku": "XYZ",
      "quantity": 1,
      "price": {"value": 30.00, "currency": "USD"}
    }
  ],
  "total": {"value": 80.00, "currency": "USD"}
}
```

**Refund request:**

```javascript
POST /api/ucp/order/refund
{
  "order_id": "ord_123",
  "refund_type": "partial",
  "items": [
    {
      "item_id": "item_1",
      "quantity": 1,
      "reason": "defective"
    }
  ],
  "refund_amount": {"value": 50.00, "currency": "USD"}
}
```

**Challenges:**

**1. Shipping/tax refund:**
Should shipping be refunded if one item is returned?

```json
{
  "refund_breakdown": {
    "items": {"value": 50.00, "currency": "USD"},
    "shipping": {"value": 0.00, "currency": "USD"},  // Not refunded
    "tax": {"value": 4.00, "currency": "USD"},       // Prorated
    "total": {"value": 54.00, "currency": "USD"}
  }
}
```

**2. Payment method no longer available:**
User paid with Google Pay token (single-use). How to refund?

```javascript
// PSP must refund to original card
const refundResult = await psp.refund({
  original_transaction_id: 'txn_abc123',
  amount: {value: 54.00, currency: 'USD'},
  reason: 'partial_refund'
});

// PSP uses network token to refund
// User sees credit on same card statement
```

**3. Multiple payment methods:**
User paid $50 with gift card, $30 with credit card. Refund $50 item.

```json
{
  "refund_allocations": [
    {
      "payment_method": "gift_card",
      "payment_id": "pay_1",
      "amount": {"value": 50.00, "currency": "USD"}
    },
    {
      "payment_method": "credit_card",
      "payment_id": "pay_2",
      "amount": {"value": 0.00, "currency": "USD"}
    }
  ]
}
```

Refund goes back to gift card only.

**4. Partial refund with restocking fee:**

```json
{
  "refund_amount": {"value": 45.00, "currency": "USD"},
  "adjustments": [
    {
      "type": "restocking_fee",
      "amount": {"value": -5.00, "currency": "USD"},
      "reason": "Non-defective return"
    }
  ],
  "final_refund": {"value": 45.00, "currency": "USD"}
}
```

**Key principle**: Refunds should be **itemized** (clear what's being refunded), **transparent** (user knows deductions), and **return to original payment method** when possible.

---

## Implementation Challenges

### Testing UCP Implementations

**Q: How do you test a UCP implementation without access to real PSPs and CPs? Are there sandbox environments?**

A: Testing strategy:

**1. Mock PSP/CP for development:**

```javascript
// Mock Credential Provider
class MockCP {
  tokenize(cardDetails) {
    return {
      token: `mock_tok_${Date.now()}`,
      last4: cardDetails.number.slice(-4),
      brand: 'visa',
      expires_at: new Date(Date.now() + 3600000).toISOString()
    };
  }

  exchangeToken(token, psp) {
    // Mock token exchange
    return {
      success: true,
      payment_method: {
        type: 'card',
        last4: '4242',
        brand: 'visa'
      }
    };
  }
}

// Mock PSP
class MockPSP {
  async authorize(payment) {
    // Simulate different scenarios based on token
    if (payment.token.includes('requires_action')) {
      return {
        status: 'requires_action',
        action: {
          type: 'ap2_mandate',
          mandate_id: `mock_mandate_${Date.now()}`,
          challenge_url: 'https://mock-psp.com/3ds/challenge'
        }
      };
    }

    if (payment.token.includes('insufficient_funds')) {
      return {
        status: 'failed',
        error: 'insufficient_funds'
      };
    }

    return {
      status: 'authorized',
      authorization_id: `auth_${Date.now()}`
    };
  }

  async capture(authorizationId) {
    return {
      status: 'captured',
      transaction_id: `txn_${Date.now()}`
    };
  }
}
```

**2. Conformance test suite:**

```bash
# Clone conformance tests
git clone https://github.com/Universal-Commerce-Protocol/conformance

# Run against your implementation
npm install
npm test -- \
  --endpoint=https://your-business.test/api/ucp \
  --profile=https://your-business.test/.well-known/ucp \
  --mock-payments=true
```

Conformance tests cover:
- Profile validation
- Checkout create/update/complete flows
- Error handling
- Webhook signature verification
- Schema validation

**3. Sandbox PSPs:**

Major PSPs provide sandbox environments:

```javascript
// Stripe test mode
const stripe = new Stripe('sk_test_...', {apiVersion: '2026-01-15'});

// Test card numbers
const testCards = {
  success: '4242424242424242',
  requires_action: '4000002500003155',
  declined: '4000000000000002'
};
```

**4. Local testing with ngrok:**

```bash
# Expose local server for webhook testing
ngrok http 3000

# Use ngrok URL in profile
{
  "webhooks": {
    "endpoint": "https://abc123.ngrok.io/webhooks/ucp"
  }
}
```

**5. Contract testing:**

```javascript
// Pact contract test
const { Pact } = require('@pact-foundation/pact');

const provider = new Pact({
  consumer: 'Platform',
  provider: 'Business',
  port: 8080
});

// Define expected interactions
await provider.addInteraction({
  state: 'business has product ABC in stock',
  uponReceiving: 'a checkout create request',
  withRequest: {
    method: 'POST',
    path: '/api/ucp/checkout/create',
    body: {
      items: [{sku: 'ABC', quantity: 1}]
    }
  },
  willRespondWith: {
    status: 200,
    body: {
      checkout_id: Matchers.uuid(),
      items: Matchers.eachLike({
        sku: 'ABC',
        quantity: 1,
        price: Matchers.decimal(99.99)
      })
    }
  }
});
```

**Key principle**: Use **mocks for development**, **conformance tests for validation**, **sandbox PSPs for integration**, and **contract tests for API contracts**.

---

**Last Updated**: January 2026 | **Version**: 2026-01-11

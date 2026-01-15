# UCP Learning Journey

**Version:** 2026-01-15
**Purpose:** A structured learning path to understand and implement the Universal Commerce Protocol

---

## Part 1: Foundations - Understanding the Why

### 1.1 What Problem Does UCP Solve?

**The Current Reality:**
Today's commerce landscape is fragmented. Every platform that wants to enable commerce (AI agents, apps, social media) needs to build custom integrations with each business. A platform supporting 100 businesses needs 100 different integrations. This doesn't scale.

**The UCP Solution:**
UCP provides a standardized common language - like HTTP for commerce. Instead of building custom integrations, everyone speaks the same protocol.

**Key Benefits:**
- **For Platforms:** One integration works with all UCP-compliant businesses
- **For Businesses:** Expose commerce capabilities once, be discovered by any platform
- **For Users:** Seamless commerce across any surface (chat, voice, visual)
- **For Payment Providers:** Standardized, secure payment flows

**Real-World Analogy:**
Think of electrical outlets. Before standardization, every appliance needed its own custom socket. UCP is that standardization for commerce - one "socket" that works everywhere.

**Doc Links:**
- [UCP Overview](https://ucp.dev/specification/overview/)

---

### 1.2 The Four Actors Model

UCP defines four distinct roles in any commerce transaction. Understanding these actors is fundamental.

#### Platform (Application/Agent)
- **Role:** Consumer-facing surface acting on behalf of users
- **Responsibilities:**
  - Discovering business capabilities
  - Orchestrating checkout flows
  - Presenting UI or conversational interface
- **Examples:** AI Shopping Assistants, Super Apps, Search Engines, Social Commerce

#### Business (Merchant)
- **Role:** Seller of goods/services, always the Merchant of Record (MoR)
- **Responsibilities:**
  - Exposing commerce capabilities (inventory, pricing, tax)
  - Fulfilling orders
  - Processing payments via their PSP
- **Examples:** Retailers, Airlines, Hotels, Service Providers
- **Key Point:** Business retains financial liability and customer relationship

#### Credential Provider (CP)
- **Role:** Manages sensitive user data securely
- **Responsibilities:**
  - Authenticating users
  - Issuing payment tokens (raw card data never touches Platform/Business)
  - Holding PII to minimize compliance scope
- **Examples:** Digital Wallets (Google Wallet, Apple Pay), Identity Providers

#### Payment Service Provider (PSP)
- **Role:** Financial infrastructure provider
- **Responsibilities:**
  - Authorizing and capturing transactions
  - Handling settlements with card networks
  - Exchanging tokens with Credential Providers
- **Examples:** Stripe, Adyen, PayPal, Braintree

**Why This Separation Matters:**
1. **Security:** Raw payment data never touches Platform or Business
2. **Compliance:** Reduces PCI scope for Platform and Business
3. **Flexibility:** Mix and match any Platform + Business + PSP/CP
4. **Separation of Concerns:** Each actor focuses on their expertise

**Doc Links:**
- [Core Concepts - Roles](https://ucp.dev/documentation/core-concepts/)

---

### 1.3 How UCP Differs from Existing Standards

**Common Question:** "We already have Payment Request API, Apple Pay, Google Pay. Why UCP?"

**The Key Difference:**
- **Payment Request API / Apple Pay / Google Pay:** Focus on payment instrument collection and tokenization (the CP role in UCP)
- **UCP:** Provides the full commerce lifecycle

**What UCP Covers:**
1. Product discovery
2. Cart management
3. Tax calculation
4. Checkout orchestration
5. Order management
6. Post-purchase support (returns, tracking)
7. Webhooks for real-time updates

**Complementary, Not Competitive:**
UCP actually *uses* existing payment standards. Example flow:
1. Platform initiates UCP checkout
2. Business calculates tax/shipping via UCP
3. Payment collected via Google Pay (which issues a token)
4. Business's PSP exchanges token with Google to process payment
5. Order updates flow through UCP webhooks

**Key Insight:** UCP is higher-level - it's about the entire commerce interaction, not just payments.

**Doc Links:**
- [UCP and AP2](https://ucp.dev/documentation/ucp-and-ap2/)

---

## Part 2: Core Concepts - The Building Blocks

### 2.1 Capabilities, Extensions, and Services

UCP is built on three fundamental constructs:

#### Capabilities
**Definition:** Standalone core features that a business supports. These are the "verbs" of the protocol.

**Core Capabilities:**
- `dev.ucp.shopping.checkout` - Complete checkout session management
- `dev.ucp.common.identity_linking` - OAuth 2.0 based identity
- `dev.ucp.shopping.order` - Order lifecycle management

**Format:** `{reverse-domain}.{service}.{capability}`

#### Extensions
**Definition:** Optional capabilities that augment another capability.

**How They Work:**
- Use the `extends` field to declare parent capability
- Appear in `ucp.capabilities[]` alongside core capabilities
- Add new fields or modify shared structures

**Examples:**
- `dev.ucp.shopping.discount` (extends checkout) - Adds discount handling
- `dev.ucp.shopping.fulfillment` (extends checkout) - Adds shipping/pickup options
- `com.example.loyalty` (extends checkout) - Vendor-specific loyalty program

#### Services
**Definition:** Lower-level communication layers for data exchange.

**Transport Options:**
- **REST API** (primary) - OpenAPI 3.x spec
- **MCP** (Model Context Protocol) - OpenRPC spec
- **A2A** (Agent2Agent) - Agent Card spec
- **Embedded Protocol** - OpenRPC for iframe communication

**Doc Links:**
- [Specification Overview](https://ucp.dev/specification/overview/)

---

### 2.2 Discovery and Negotiation

**How It Works:**

#### 1. Platform Profile
Platform declares what it supports:
```json
{
  "ucp": {
    "version": "2026-01-11",
    "capabilities": [
      "dev.ucp.shopping.checkout",
      "dev.ucp.common.identity_linking"
    ]
  }
}
```

#### 2. Business Profile
Business declares what it offers via `/.well-known/ucp-profile.json`:
```json
{
  "ucp": {
    "version": "2026-01-11",
    "capabilities": [
      {
        "name": "dev.ucp.shopping.checkout",
        "version": "2026-01-11",
        "spec": "https://ucp.dev/specification/checkout",
        "schema": "https://ucp.dev/schemas/shopping/checkout.json",
        "service": {
          "version": "2026-01-11",
          "rest": {
            "schema": "https://ucp.dev/services/shopping/rest.openapi.json",
            "endpoint": "https://business.example.com/api/v2"
          }
        }
      }
    ]
  }
}
```

#### 3. Negotiation (Server-Selects)
The business (server) chooses the version from the intersection of capabilities:
1. Platform says "I support versions X, Y, Z"
2. Business says "I support versions Y, Z, W"
3. Business selects Y or Z (usually latest both support)
4. Business uses that version for the session

**Benefits of Server-Selects:**
- Business controls rollout
- Easier A/B testing
- Simpler platform code (no fallback chains)
- Gradual evolution without breaking changes

**Doc Links:**
- [Specification Overview - Discovery](https://ucp.dev/specification/overview/#discovery-governance-and-negotiation)

---

### 2.3 Governance Without Central Authority

**The Challenge:** How do you govern an open protocol without a central registry?

**UCP Solution:** Reverse-domain namespacing + DNS-based authority

#### Namespace Format
```
{reverse-domain}.{service}.{capability}
```

**Examples:**
- `dev.ucp.shopping.checkout` - Official UCP capability
- `com.shopify.payments.installments` - Shopify-specific extension
- `com.example.loyalty.rewards` - Example vendor capability

#### Security Binding
The `spec` and `schema` URLs **MUST** have origins matching the namespace authority:

| Namespace | Required Origin |
|-----------|----------------|
| `dev.ucp.*` | `https://ucp.dev/...` |
| `com.shopify.*` | `https://shopify.com/...` |
| `com.example.*` | `https://example.com/...` |

**Why This Matters:**
- Prevents capability hijacking
- Domain ownership proves authority
- DNS is the governance mechanism
- No central registry needed

**Doc Links:**
- [Specification Overview - Namespace Governance](https://ucp.dev/specification/overview/#namespace-governance)

---

## Part 3: Checkout - The Core Capability

### 3.1 Checkout Session Lifecycle

A checkout session goes through several states:

```
created → ready_for_complete → completing → completed
                             ↓
                          cancelled
```

#### State Descriptions:
- **created:** Session initialized, buyer can modify items/address
- **ready_for_complete:** All data collected, ready for payment
- **completing:** Payment processing in progress
- **completed:** Order created successfully
- **cancelled:** Session abandoned or failed

**Doc Links:**
- [Checkout Specification](https://ucp.dev/specification/checkout/)

---

### 3.2 REST Checkout Flow (Platform-Driven UI)

**When to Use:** Platform wants full control over checkout UI and experience.

#### Step-by-Step Flow:

**Step 1: Create Session**
```bash
POST https://business.example.com/api/v2/checkout-sessions
```

Request:
```json
{
  "ucp": {
    "version": "2026-01-11",
    "capabilities": ["dev.ucp.shopping.checkout"]
  },
  "buyer": {
    "email": "user@example.com"
  },
  "line_items": [
    {
      "item": {
        "id": "item_123",
        "title": "Wireless Headphones",
        "price": 9999
      },
      "quantity": 1
    }
  ]
}
```

Response includes checkout ID and current state.

**Step 2: Update Session (as user adds shipping, etc.)**
```bash
PATCH https://business.example.com/api/v2/checkout-sessions/{id}
```

**Step 3: Complete Checkout**
```bash
POST https://business.example.com/api/v2/checkout-sessions/{id}/complete
```

With payment handler result from AP2 flow.

**Doc Links:**
- [REST Checkout Spec](https://ucp.dev/specification/checkout-rest/)

---

### 3.3 Embedded Checkout Flow (Business-Driven UI)

**When to Use:** Business needs to handle complex checkout logic (subscriptions, bundles, promotions) with their own UI.

#### How It Works:

**Step 1: Platform Creates Session**
```bash
POST /checkout-sessions
```

Response includes `embedded_url`.

**Step 2: Platform Embeds in iframe**
```html
<iframe src="{embedded_url}" />
```

**Step 3: Bidirectional Communication**
Platform and Business communicate via postMessage:

From Business → Platform:
```json
{
  "type": "delegatePayment",
  "data": {
    "amount": 9999,
    "currency": "USD"
  }
}
```

From Platform → Business:
```json
{
  "type": "paymentResult",
  "data": {
    "paymentHandler": {...}
  }
}
```

**Key Features:**
- Business controls UI and validation
- Platform handles payment collection
- Real-time communication during checkout
- Complex flows (subscriptions, tax exemptions)

**Doc Links:**
- [Embedded Checkout Spec](https://ucp.dev/specification/embedded-checkout/)

---

### 3.4 MCP Checkout Flow (Agent-Driven)

**When to Use:** AI agents acting on behalf of users, conversational commerce.

**How It Works:**
Agents call UCP functions via Model Context Protocol:

```typescript
await mcpClient.call('create_checkout_session', {
  buyer: { email: 'user@example.com' },
  line_items: [...]
});
```

**Key Differences:**
- Natural language→structured calls
- No traditional UI
- Agent makes decisions based on user intent
- Uses same UCP data models as REST

**Doc Links:**
- [MCP Checkout Spec](https://ucp.dev/specification/checkout-mcp/)

---

## Part 4: Transport Protocols

### 4.1 REST API (Primary Transport)

**Design Principles:**
- RESTful patterns (GET, POST, PATCH)
- JSON payloads
- HTTPS required
- OpenAPI 3.x specifications

**Common Patterns:**

#### Resource Creation:
```
POST /{resource}
Returns: 201 Created with resource
```

#### Resource Update:
```
PATCH /{resource}/{id}
Returns: 200 OK with updated resource
```

#### Resource Retrieval:
```
GET /{resource}/{id}
Returns: 200 OK with resource
```

**Error Handling:**
```json
{
  "error": {
    "code": "validation_error",
    "message": "Invalid email address",
    "param": "buyer.email"
  }
}
```

**Doc Links:**
- [REST Specifications](https://ucp.dev/specification/checkout-rest/)

---

### 4.2 Model Context Protocol (MCP)

**Purpose:** Enable AI agents to discover and invoke UCP capabilities as tools.

**How It Works:**
1. Agent discovers UCP server via MCP protocol
2. Server exposes UCP operations as callable tools
3. Agent invokes tools to complete commerce tasks

**Example Tool Definition:**
```json
{
  "name": "create_checkout_session",
  "description": "Create a new checkout session",
  "inputSchema": {
    "$ref": "https://ucp.dev/schemas/shopping/checkout_create_req.json"
  }
}
```

**Benefits:**
- Agents can "learn" commerce capabilities
- Same data models as REST
- Optimized for conversational flows

**Doc Links:**
- [MCP Checkout Spec](https://ucp.dev/specification/checkout-mcp/)
- [Model Context Protocol](https://modelcontextprotocol.io/)

---

### 4.3 Agent2Agent Protocol (A2A)

**Purpose:** Peer-to-peer agent communication for commerce.

**Key Concepts:**
- Agents publish Agent Cards (capability declarations)
- Other agents discover and invoke capabilities
- Decentralized, no central broker

**Example Agent Card:**
```json
{
  "id": "business-agent-123",
  "name": "Acme Store Agent",
  "capabilities": [
    "dev.ucp.shopping.checkout"
  ],
  "methods": {
    "create_checkout": {
      "description": "...",
      "parameters": {...}
    }
  }
}
```

**Use Cases:**
- Agent-to-agent negotiations
- Autonomous commerce
- Multi-agent orchestration

**Doc Links:**
- [A2A Checkout Spec](https://ucp.dev/specification/checkout-a2a/)
- [A2A Protocol](https://a2a-protocol.org/)

---

## Part 5: Security & Privacy

### 5.1 Identity Linking (OAuth 2.0)

**Purpose:** Secure, authorized relationships between Platforms and Businesses without sharing credentials.

**Standard:** OAuth 2.0 Authorization Code Flow

**Scope Format:**
```
ucp:scopes:{capability_name}
```

**Example Flow:**

**Step 1: Authorization Request**
```
GET https://business.example.com/oauth2/authorize?
  response_type=code&
  client_id=platform_client_id&
  scope=ucp:scopes:checkout_session&
  state=random_state
```

**Step 2: User Authorizes**
Business authenticates user and shows consent screen.

**Step 3: Token Exchange**
```bash
POST https://business.example.com/oauth2/token
```

```json
{
  "grant_type": "authorization_code",
  "code": "auth_code_123",
  "client_id": "platform_client_id",
  "client_secret": "secret"
}
```

**Step 4: Use Access Token**
```bash
GET /checkout-sessions
Authorization: Bearer {access_token}
```

**Benefits:**
- Standard, well-understood protocol
- User controls what Platforms can access
- Tokens can be revoked
- No credential sharing

**Doc Links:**
- [Identity Linking Spec](https://ucp.dev/specification/identity-linking/)

---

### 5.2 Payment Security (AP2 Integration)

**Agent Payments Protocol (AP2):** A verifiable credential system for secure, provable payments.

**Key Concepts:**
- **Payment Mandates:** Cryptographically signed authorizations
- **Payment Handlers:** Execute mandates (can be Business or Platform)
- **Verifiable Credentials:** Proof of user consent

**Payment Flow with AP2:**

1. **Platform collects payment intent from user**
2. **CP (e.g., Google Wallet) issues payment mandate:**
```json
{
  "type": "payment_mandate",
  "amount": 9999,
  "currency": "USD",
  "credential": {
    "proof": "cryptographic_signature"
  }
}
```
3. **Payment handler uses mandate with PSP**
4. **PSP verifies signature and processes payment**

**Security Benefits:**
- Cryptographic proof of user consent
- No raw card data exposed to Platform/Business
- PSP can independently verify authorization
- Prevents unauthorized charges

**Doc Links:**
- [UCP and AP2](https://ucp.dev/documentation/ucp-and-ap2/)
- [AP2 Mandates Spec](https://ucp.dev/specification/ap2-mandates/)
- [AP2 Protocol](https://ap2-protocol.org/)

---

### 5.3 Payment Handler Patterns

Three common patterns for payment processing:

#### Pattern 1: Business as Payment Handler
- Business owns PSP relationship
- Business receives payment mandate
- Business submits to their PSP

**Pros:** Full control, existing PSP relationships
**Cons:** Business handles payment complexity

#### Pattern 2: Platform as Payment Handler
- Platform facilitates payment on behalf of Business
- Platform receives mandate, submits to Business's PSP
- Business remains MoR

**Pros:** Simplified for Business, Platform handles payment UX
**Cons:** Platform needs PSP integrations

#### Pattern 3: Encrypted Credentials
- CP encrypts payment credentials for Business's PSP
- Only PSP can decrypt
- Credentials never exposed to Platform or Business

**Pros:** Maximum security, minimal scope
**Cons:** Requires CP-PSP integration

**Doc Links:**
- [Payment Handler Guide](https://ucp.dev/specification/payment-handler-guide/)
- [Example Implementations](https://ucp.dev/specification/examples/business-tokenizer-payment-handler/)

---

## Part 6: Order Management

### 6.1 Order Lifecycle

After checkout completes, the order enters post-purchase lifecycle:

```
created → fulfilling → fulfilled
                    ↓
                  cancelled/refunded
```

**Order Object Contains:**
- Link to original checkout
- Line items purchased
- Fulfillment expectations (when items will arrive)
- Fulfillment events (shipped, delivered)
- Adjustments (refunds, cancellations)
- Current totals

---

### 6.2 Fulfillment Tracking

**Fulfillment Expectations:**
Tell buyer when/how items will arrive:
```json
{
  "id": "exp_1",
  "line_items": [{"id": "li_1", "quantity": 1}],
  "method_type": "shipping",
  "destination": {
    "full_name": "Jane Doe",
    "street_address": "123 Main St",
    ...
  },
  "description": "Arrives in 2-3 business days",
  "fulfillable_on": "2026-01-20T00:00:00Z"
}
```

**Fulfillment Events:**
Track actual fulfillment:
```json
{
  "id": "evt_1",
  "occurred_at": "2026-01-20T14:30:00Z",
  "type": "shipped",
  "line_items": [{"id": "li_1", "quantity": 1}],
  "tracking_number": "1Z999AA10123456784",
  "tracking_url": "https://www.ups.com/track?id=1Z999AA10123456784"
}
```

**Event Types:**
- `shipped` - Package left facility
- `in_transit` - En route to destination
- `out_for_delivery` - On delivery vehicle
- `delivered` - Received by customer
- `failed_delivery` - Delivery unsuccessful
- `ready_for_pickup` - Available at pickup location
- `picked_up` - Collected by customer

---

### 6.3 Adjustments (Refunds, Cancellations)

**Refund Example:**
```json
{
  "id": "adj_1",
  "type": "refund",
  "occurred_at": "2026-01-22T10:00:00Z",
  "status": "completed",
  "line_items": [{"id": "li_1", "quantity": 1}],
  "amount": 9999,
  "description": "Customer requested refund for defective item"
}
```

**Adjustment Types:**
- `refund` - Money returned to customer
- `cancellation` - Order cancelled before fulfillment
- `partial_cancellation` - Some items cancelled
- `price_adjustment` - Price correction

**Doc Links:**
- [Order Specification](https://ucp.dev/specification/order/)

---

### 6.4 Webhooks for Real-Time Updates

**Purpose:** Notify Platform of order changes in real-time.

**How It Works:**

**Step 1: Platform Registers Webhook**
```json
{
  "ucp": {...},
  "webhook_endpoint": "https://platform.example.com/webhooks/ucp",
  "events": ["order.updated", "order.fulfilled"]
}
```

**Step 2: Business Sends Updates**
```bash
POST https://platform.example.com/webhooks/ucp
```

```json
{
  "event": "order.updated",
  "occurred_at": "2026-01-20T14:30:00Z",
  "order": {
    "id": "order_123",
    "fulfillment": {
      "events": [
        {
          "type": "shipped",
          "tracking_number": "...",
          ...
        }
      ]
    }
  }
}
```

**Webhook Security:**
- HTTPS required
- Signature verification (HMAC)
- Retry with exponential backoff
- Idempotency keys

**Doc Links:**
- [Order Specification - Webhooks](https://ucp.dev/specification/order/)

---

## Part 7: Extensions & Customization

### 7.1 Discount Extensions

**Purpose:** Support promotional pricing, coupons, loyalty points.

**Schema Additions:**
Adds `discount` to `totals.type`:
```json
{
  "totals": [
    {"type": "subtotal", "amount": 15000},
    {"type": "discount", "amount": -2000, "title": "SAVE20"},
    {"type": "shipping", "amount": 500},
    {"type": "tax", "amount": 1050},
    {"type": "total", "amount": 14550}
  ]
}
```

**Discount Application:**
```json
{
  "applied_discounts": [
    {
      "code": "SAVE20",
      "type": "percentage",
      "value": 20,
      "amount": 2000,
      "line_item_ids": ["li_1", "li_2"]
    }
  ]
}
```

**Doc Links:**
- [Discount Specification](https://ucp.dev/specification/discount/)

---

### 7.2 Fulfillment Extensions

**Purpose:** Advanced shipping/pickup options.

**Schema Additions:**
Adds fulfillment object to checkout:
```json
{
  "fulfillment": {
    "methods": [
      {
        "id": "method_1",
        "type": "shipping",
        "line_item_ids": ["li_1"],
        "selected_destination_id": "dest_1",
        "destinations": [
          {
            "id": "dest_1",
            "full_name": "Jane Doe",
            "street_address": "123 Main St",
            "address_locality": "San Francisco",
            "address_region": "CA",
            "postal_code": "94102",
            "address_country": "US"
          }
        ],
        "groups": [
          {
            "id": "group_1",
            "line_item_ids": ["li_1"],
            "selected_option_id": "standard",
            "options": [
              {
                "id": "standard",
                "title": "Standard Shipping (5-7 days)",
                "totals": [{"type": "shipping", "amount": 500}]
              },
              {
                "id": "express",
                "title": "Express Shipping (2-3 days)",
                "totals": [{"type": "shipping", "amount": 1500}]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**Method Types:**
- `shipping` - Delivery to address
- `pickup` - Customer picks up
- `digital` - Digital delivery (email, download)

**Doc Links:**
- [Fulfillment Specification](https://ucp.dev/specification/fulfillment/)

---

### 7.3 Creating Custom Extensions

**When to Create:**
- Your business needs capabilities not in core UCP
- Industry-specific features (subscriptions, rentals)
- Vendor-specific services (loyalty programs)

**Steps:**

**Step 1: Define Namespace**
```
com.yourcompany.{service}.{capability}
```

**Step 2: Create Schema**
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "https://yourcompany.com/schemas/loyalty.json",
  "title": "Loyalty Extension",
  "description": "Loyalty points and rewards",
  "extends": "dev.ucp.shopping.checkout",
  "definitions": {
    "loyalty_points": {
      "type": "object",
      "properties": {
        "available": {"type": "integer"},
        "earned": {"type": "integer"},
        "redeemed": {"type": "integer"}
      }
    }
  },
  "allOf": [
    {
      "$ref": "https://ucp.dev/schemas/shopping/checkout.json"
    },
    {
      "properties": {
        "loyalty": {
          "$ref": "#/definitions/loyalty_points"
        }
      }
    }
  ]
}
```

**Step 3: Publish Spec**
Host at `https://yourcompany.com/specs/loyalty`

**Step 4: Declare in Profile**
```json
{
  "ucp": {
    "capabilities": [
      {
        "name": "com.yourcompany.shopping.loyalty",
        "version": "2026-01-15",
        "spec": "https://yourcompany.com/specs/loyalty",
        "schema": "https://yourcompany.com/schemas/loyalty.json",
        "extends": "dev.ucp.shopping.checkout"
      }
    ]
  }
}
```

**Doc Links:**
- [Schema Authoring Guide](https://ucp.dev/documentation/schema-authoring/)

---

## Part 8: Implementation Guide

### 8.1 For Businesses: Implementing UCP

**Step 1: Decide Your Capabilities**
Choose what you'll support:
- ✅ `dev.ucp.shopping.checkout` (required)
- ✅ `dev.ucp.common.identity_linking` (recommended)
- ✅ `dev.ucp.shopping.order` (recommended)
- ✅ Extensions: discount, fulfillment, etc.

**Step 2: Choose Transport(s)**
- REST API (required for most integrations)
- MCP (if targeting AI agents)
- A2A (if building agent capabilities)
- Embedded (if need complex checkout UI)

**Step 3: Implement Core APIs**
```
POST /checkout-sessions
PATCH /checkout-sessions/{id}
POST /checkout-sessions/{id}/complete
GET /orders/{id}
```

**Step 4: Publish Profile**
Host `/.well-known/ucp-profile.json`

**Step 5: Test with Platforms**
Use UCP Playground or partner with platforms

**Doc Links:**
- [Implementation Guide (coming soon)](https://ucp.dev/)
- [Code Samples](https://github.com/Universal-Commerce-Protocol/samples)

---

### 8.2 For Platforms: Consuming UCP

**Step 1: Discover Businesses**
Fetch `https://business.example.com/.well-known/ucp-profile.json`

**Step 2: Negotiate Capabilities**
```typescript
const platformCaps = ['dev.ucp.shopping.checkout'];
const businessCaps = profile.ucp.capabilities.map(c => c.name);
const supported = platformCaps.filter(c => businessCaps.includes(c));
```

**Step 3: Fetch and Compose Schemas**
```typescript
// Fetch base schema
const checkoutSchema = await fetch(
  'https://ucp.dev/schemas/shopping/checkout.json'
).then(r => r.json());

// Fetch extension schemas
const extensions = profile.ucp.capabilities
  .filter(c => c.extends === 'dev.ucp.shopping.checkout')
  .map(c => fetch(c.schema).then(r => r.json()));

// Compose schemas
const fullSchema = composeSchemas(checkoutSchema, ...extensions);
```

**Step 4: Make API Calls**
```typescript
const session = await createCheckoutSession({
  ucp: {
    version: selectedVersion,
    capabilities: supported
  },
  buyer: {...},
  line_items: [...]
});
```

**Step 5: Handle Webhooks**
Listen for order updates and refresh UI

**Doc Links:**
- [Platform Integration Guide (coming soon)](https://ucp.dev/)

---

### 8.3 For Payment Providers: PSP Integration

**As a PSP, you integrate with:**
1. **Businesses** (your existing relationship)
2. **Credential Providers** (token exchange)

**Token Exchange Flow:**
```typescript
// Receive payment mandate from Business
const mandate = {
  type: 'payment_mandate',
  amount: 9999,
  credential: {
    token: 'tok_google_wallet_123',
    proof: 'signature...'
  }
};

// Exchange token with CP (e.g., Google)
const payment = await exchangeToken({
  token: mandate.credential.token,
  amount: mandate.amount,
  merchant_id: business.id
});

// Process payment
const result = await processPayment(payment);
```

**Key Responsibilities:**
- Verify payment mandate signatures
- Exchange tokens with CPs
- Process payments
- Return authorization results

**Doc Links:**
- [Payment Handler Guide](https://ucp.dev/specification/payment-handler-guide/)
- [Tokenization Guide](https://ucp.dev/specification/tokenization-guide/)

---

### 8.4 Testing and Validation

**UCP Playground:**
Interactive testing environment for all actors:
- Test as Platform (initiate checkouts)
- Test as Business (respond to requests)
- Test as PSP (handle payments)
- Inspect payloads and flows

**Access:** [https://ucp.dev/playground/](https://ucp.dev/playground/)

**JSON Schema Validation:**
```typescript
import Ajv from 'ajv';

const ajv = new Ajv();
const schema = await fetch('https://ucp.dev/schemas/shopping/checkout.json')
  .then(r => r.json());

const validate = ajv.compile(schema);
const valid = validate(checkoutData);

if (!valid) {
  console.error(validate.errors);
}
```

**Code Samples Repository:**
Reference implementations in multiple languages:
- Node.js / TypeScript
- Python
- Go
- Java

**Repo:** [https://github.com/Universal-Commerce-Protocol/samples](https://github.com/Universal-Commerce-Protocol/samples)

---

## Part 9: Advanced Topics

### 9.1 Schema Composition Deep Dive

**How Extensions Modify Base Types:**

Base schema (`checkout.json`):
```json
{
  "definitions": {
    "total": {
      "type": "object",
      "properties": {
        "type": {"enum": ["subtotal", "shipping", "tax", "total"]},
        "amount": {"type": "integer"}
      }
    }
  }
}
```

Extension schema (`discount.json`):
```json
{
  "allOf": [
    {"$ref": "https://ucp.dev/schemas/shopping/checkout.json"},
    {
      "definitions": {
        "total": {
          "properties": {
            "type": {
              "enum": ["discount"]  // Adds "discount" to allowed values
            }
          }
        }
      }
    }
  ]
}
```

**Platform Composition:**
```typescript
function composeSchemas(base, ...extensions) {
  const composed = JSON.parse(JSON.stringify(base));

  for (const ext of extensions) {
    // Merge definitions
    for (const [key, value] of Object.entries(ext.definitions || {})) {
      if (composed.definitions[key]) {
        // Merge enum values
        if (value.properties?.type?.enum) {
          composed.definitions[key].properties.type.enum.push(
            ...value.properties.type.enum
          );
        }
      } else {
        composed.definitions[key] = value;
      }
    }
  }

  return composed;
}
```

---

### 9.2 Version Migration Strategies

**Backward Compatibility:**
UCP versions are date-based (YYYY-MM-DD). New versions:
- **MAY** add new optional fields
- **MUST NOT** remove required fields
- **MUST NOT** change field semantics

**Migration Pattern:**
```typescript
// Support multiple versions
const handlers = {
  '2026-01-11': handleV20260111,
  '2026-02-15': handleV20260215
};

const version = request.ucp.version;
const handler = handlers[version] || handlers['2026-01-11'];
await handler(request);
```

**Deprecation Policy:**
- Versions supported for minimum 12 months
- Deprecation notices 6 months in advance
- Critical fixes backported to all supported versions

---

### 9.3 Multi-Currency Support

**Currency in UCP:**
- Always specified in checkout/order
- Amount in minor units (cents)
- ISO 4217 currency codes

**Example:**
```json
{
  "currency": "USD",
  "totals": [
    {"type": "subtotal", "amount": 10000},  // $100.00
    {"type": "tax", "amount": 800},         // $8.00
    {"type": "total", "amount": 10800}      // $108.00
  ]
}
```

**Multi-Currency Business:**
```typescript
// Business determines currency based on:
// - Buyer location
// - Platform preference
// - Business's supported currencies

const currency = determineCurrency(buyer, platform);
const prices = await getPrices(items, currency);
```

---

### 9.4 Error Handling Best Practices

**Standard Error Format:**
```json
{
  "error": {
    "code": "validation_error",
    "message": "The email address is invalid",
    "param": "buyer.email",
    "details": {
      "reason": "Email must contain @ symbol"
    }
  }
}
```

**Common Error Codes:**
- `validation_error` - Invalid input data
- `authentication_required` - OAuth token needed
- `permission_denied` - Insufficient scope
- `resource_not_found` - Checkout/order doesn't exist
- `rate_limit_exceeded` - Too many requests
- `service_unavailable` - Temporary outage

**Retry Strategy:**
```typescript
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (!isRetryable(error) || i === maxRetries - 1) {
        throw error;
      }
      await sleep(Math.pow(2, i) * 1000);  // Exponential backoff
    }
  }
}

function isRetryable(error) {
  return [
    'rate_limit_exceeded',
    'service_unavailable',
    'timeout'
  ].includes(error.code);
}
```

---

## Part 10: Real-World Use Cases

### 10.1 AI Shopping Assistant

**Scenario:** User asks AI "Find me wireless headphones under $100"

**Flow:**
1. **Discovery:** AI searches for businesses selling headphones
2. **Filter:** Check `.well-known/ucp-profile.json` for UCP support
3. **Query:** Use MCP to search inventory
```typescript
const results = await mcp.call('search_products', {
  query: 'wireless headphones',
  max_price: 10000  // $100.00 in cents
});
```
4. **Present:** Show options to user in conversation
5. **Checkout:** User selects, AI creates checkout session via MCP
6. **Payment:** AI delegates to user's credential provider
7. **Confirm:** Order created, tracking info sent to user

**Benefits:**
- Natural language → structured commerce
- One integration works across all UCP businesses
- User controls payment via their wallet

---

### 10.2 Social Commerce Platform

**Scenario:** Social media app wants to enable in-app shopping

**Implementation:**
1. **Discovery:** Crawl business profiles for UCP support
2. **Product Showcase:** Display products with UCP metadata
3. **Native Checkout:** Use REST API for checkout in-app
```typescript
// User taps "Buy Now" on influencer post
const session = await createCheckoutSession({
  line_items: [productFromPost],
  buyer: {email: currentUser.email}
});

// Show native checkout UI
showCheckoutUI(session);
```
4. **Payment:** Collect via platform's payment handler
5. **Tracking:** Display order status via webhooks

**Benefits:**
- Seamless in-app experience
- Business remains MoR (no marketplace complications)
- Revenue from payment facilitation

---

### 10.3 Voice Commerce

**Scenario:** Smart speaker shopping

**Flow:**
1. **Voice Command:** "Alexa, order coffee beans"
2. **Intent Recognition:** Parse into structured request
3. **Business Selection:** User's preferred business or best match
4. **Checkout via Voice:**
```typescript
await voiceAgent.mcp.call('create_checkout_session', {
  buyer: {email: userProfile.email},
  line_items: [coffeeProduct]
});

// Confirm with user
await speak("That'll be $24.99 with free shipping. Confirm?");
await listen();  // "Yes"

// Complete
await voiceAgent.mcp.call('complete_checkout', {
  session_id: session.id,
  payment_handler: userWallet
});
```

**Benefits:**
- Hands-free shopping
- Consistent across products/businesses
- Secure payment via user's wallet

---

### 10.4 Embedded Marketplace

**Scenario:** Blog platform wants affiliate commerce

**Implementation:**
```html
<!-- Blogger embeds product -->
<div data-ucp-product="https://business.example.com/products/123">
  <button onclick="buyProduct()">Buy Now</button>
</div>

<script>
async function buyProduct() {
  // Create checkout session
  const session = await fetch('https://business.example.com/api/checkout-sessions', {
    method: 'POST',
    body: JSON.stringify({
      ucp: {capabilities: ['dev.ucp.shopping.checkout']},
      line_items: [{item: {id: '123'}, quantity: 1}]
    })
  }).then(r => r.json());

  // Show embedded checkout
  const modal = document.createElement('iframe');
  modal.src = session.embedded_url;
  document.body.appendChild(modal);
}
</script>
```

**Benefits:**
- No custom Business integrations
- Blogger earns affiliate commission
- User stays on blog (better conversion)

---

## Part 11: Best Practices

### 11.1 Security Checklist

**For All Actors:**
- [ ] Always use HTTPS
- [ ] Validate all input data
- [ ] Implement rate limiting
- [ ] Log security events
- [ ] Keep dependencies updated

**For Platforms:**
- [ ] Validate spec URL origins match namespaces
- [ ] Fetch and compose schemas client-side
- [ ] Verify webhook signatures
- [ ] Implement OAuth securely
- [ ] Never log/store raw payment data

**For Businesses:**
- [ ] Implement OAuth correctly (state param, PKCE)
- [ ] Sign webhook payloads
- [ ] Validate payment handler results
- [ ] Use PCI-compliant PSPs
- [ ] Implement fraud detection

**For PSPs:**
- [ ] Verify payment mandate signatures
- [ ] Validate token authenticity with CPs
- [ ] Implement strong authentication (3DS)
- [ ] Monitor for suspicious patterns

---

### 11.2 Performance Optimization

**Caching:**
```typescript
// Cache profiles (update hourly)
const cache = new TTLCache({ttl: 3600});

async function getBusinessProfile(businessUrl) {
  const cacheKey = `profile:${businessUrl}`;
  let profile = cache.get(cacheKey);

  if (!profile) {
    profile = await fetch(`${businessUrl}/.well-known/ucp-profile.json`)
      .then(r => r.json());
    cache.set(cacheKey, profile);
  }

  return profile;
}
```

**Schema Preloading:**
```typescript
// Preload common schemas at startup
const schemaCache = new Map();

async function preloadSchemas() {
  const schemas = [
    'https://ucp.dev/schemas/shopping/checkout.json',
    'https://ucp.dev/schemas/shopping/order.json',
    'https://ucp.dev/schemas/shopping/discount.json'
  ];

  await Promise.all(
    schemas.map(async url => {
      const schema = await fetch(url).then(r => r.json());
      schemaCache.set(url, schema);
    })
  );
}
```

**Batch Operations:**
```typescript
// Fetch multiple orders in one request (if API supports)
const orders = await fetchOrdersBatch(orderIds);

// Or use parallel requests
const orders = await Promise.all(
  orderIds.map(id => fetchOrder(id))
);
```

---

### 11.3 Observability

**Structured Logging:**
```typescript
logger.info('checkout_created', {
  session_id: session.id,
  business: business.domain,
  platform: platform.id,
  capabilities: session.ucp.capabilities,
  currency: session.currency,
  total_amount: session.totals.find(t => t.type === 'total').amount
});
```

**Metrics to Track:**
- Checkout creation rate
- Completion rate (created → completed)
- Average checkout time
- Error rates by type
- API response times
- Cache hit rates

**Distributed Tracing:**
```typescript
// Propagate trace context
const headers = {
  'x-trace-id': generateTraceId(),
  'x-span-id': generateSpanId(),
  'x-parent-span-id': currentSpan.id
};

await fetch(businessEndpoint, {headers});
```

---

### 11.4 Developer Experience

**OpenAPI for Documentation:**
```yaml
openapi: 3.0.0
info:
  title: Acme Store UCP API
  version: 2026-01-11
paths:
  /checkout-sessions:
    post:
      summary: Create checkout session
      requestBody:
        $ref: 'https://ucp.dev/schemas/shopping/checkout_create_req.json'
      responses:
        '201':
          $ref: 'https://ucp.dev/schemas/shopping/checkout_resp.json'
```

**Type Safety:**
```typescript
// Generate types from schemas
import type { CheckoutSession } from './generated/ucp-types';

function createCheckout(data: CheckoutSession): Promise<CheckoutSession> {
  // TypeScript ensures correct shape
}
```

**Clear Error Messages:**
```json
{
  "error": {
    "code": "invalid_line_item",
    "message": "Line item quantity must be positive",
    "param": "line_items[0].quantity",
    "details": {
      "received": -1,
      "expected": ">= 1"
    }
  }
}
```

---

## Part 12: Roadmap and Future

### 12.1 Upcoming Capabilities

**In Development:**
- **Subscriptions:** Recurring billing and subscription management
- **Reservations:** Hotel bookings, restaurant reservations, appointments
- **Returns & Exchanges:** Automated return merchandise authorization
- **Gift Cards:** Digital gift card purchase and redemption
- **Wishlists:** Cross-platform wishlist sync

**Proposed:**
- **Quotes:** B2B quote requests and negotiations
- **Installments:** Buy-now-pay-later integrations
- **Trade-Ins:** Device trade-in valuations
- **Personalization:** User preference sharing

---

### 12.2 Community and Governance

**How to Contribute:**
1. **GitHub Discussions:** Propose new capabilities
2. **Pull Requests:** Submit schema improvements
3. **Working Groups:** Join domain-specific groups
4. **Implementation Feedback:** Share real-world learnings

**Governance Structure:**
- **Steering Committee:** Strategic direction
- **Technical Committee:** Specification decisions
- **Working Groups:** Domain expertise (payments, logistics, etc.)

**Links:**
- [Contributing Guide](https://github.com/Universal-Commerce-Protocol/ucp/blob/main/CONTRIBUTING.md)
- [GitHub Repo](https://github.com/Universal-Commerce-Protocol/ucp)

---

### 12.3 Industry Standards Alignment

**UCP Builds On:**
- **OAuth 2.0:** Identity and authorization
- **OpenAPI:** REST API specifications
- **JSON Schema:** Data validation
- **RFC 3339:** Date/time format
- **ISO 4217:** Currency codes
- **AP2:** Payment mandates
- **MCP:** Agent tool protocol
- **A2A:** Agent-to-agent communication

**Interoperability Goals:**
- Work with existing payment networks
- Compatible with major e-commerce platforms
- Integrate with AI frameworks (LangChain, AutoGen, etc.)
- Support all major programming languages

---

## Part 13: Quick Reference

### 13.1 Common Endpoints

**Business Endpoints:**
```
GET  /.well-known/ucp-profile.json
POST /checkout-sessions
GET  /checkout-sessions/{id}
PATCH /checkout-sessions/{id}
POST /checkout-sessions/{id}/complete
GET  /orders/{id}
```

**Platform Endpoints:**
```
POST /webhooks/ucp
```

---

### 13.2 Key Concepts Glossary

- **Platform:** Consumer-facing app/agent orchestrating commerce
- **Business:** Merchant selling goods/services (always MoR)
- **CP (Credential Provider):** Manages sensitive user data, issues tokens
- **PSP (Payment Service Provider):** Processes payments
- **MoR (Merchant of Record):** Legal entity responsible for transaction
- **Capability:** Core feature (e.g., checkout, order)
- **Extension:** Optional module augmenting a capability
- **Service:** Communication layer (REST, MCP, A2A, Embedded)
- **AP2:** Agent Payments Protocol for verifiable payments
- **Payment Mandate:** Cryptographically signed payment authorization
- **Payment Handler:** Entity executing payment (Business or Platform)

---

### 13.3 Response Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PATCH |
| 201 | Created | Successful POST (created resource) |
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Missing/invalid auth token |
| 403 | Forbidden | Valid token, insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | State conflict (e.g., checkout already completed) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Temporary outage |

---

### 13.4 Date/Time Format

Always use RFC 3339:
```
2026-01-15T14:30:00Z        # UTC
2026-01-15T09:30:00-05:00   # EST (with offset)
```

---

### 13.5 Amount Format

Always minor units (cents):
```json
{
  "amount": 9999,    // $99.99
  "currency": "USD"
}
```

---

## Conclusion

You've completed the UCP Learning Journey! You now understand:

✅ **The Why:** UCP solves fragmentation in commerce
✅ **The Architecture:** Four actors working in harmony
✅ **Core Capabilities:** Checkout, Identity, Orders
✅ **Transport Options:** REST, MCP, A2A, Embedded
✅ **Security:** OAuth 2.0, AP2, payment mandates
✅ **Extensions:** Discounts, fulfillment, custom capabilities
✅ **Implementation:** Best practices for all actors
✅ **Real-World Use:** AI agents, social commerce, voice shopping

**Next Steps:**
1. Explore the [UCP Playground](https://ucp.dev/playground/)
2. Review [Code Samples](https://github.com/Universal-Commerce-Protocol/samples)
3. Join the [Community](https://github.com/Universal-Commerce-Protocol/ucp/discussions)
4. Start building!

**Questions or Feedback:**
- GitHub Discussions: [https://github.com/Universal-Commerce-Protocol/ucp/discussions](https://github.com/Universal-Commerce-Protocol/ucp/discussions)
- Documentation: [https://ucp.dev/](https://ucp.dev/)

---

*This learning document is part of the Universal Commerce Protocol project. Licensed under Apache 2.0.*

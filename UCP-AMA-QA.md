# Universal Commerce Protocol (UCP) - AMA Q&A Guide

## Fundamentals

### What is UCP?

**Q: What is the Universal Commerce Protocol and why does it exist?**

A: UCP is an open standard for commerce interoperability that addresses the fragmented commerce landscape. Currently, every platform (AI agents, apps, social media) that wants to enable commerce needs custom, one-off integrations with each business. UCP provides a standardized common language and functional primitives so that platforms, businesses, Payment Service Providers (PSPs), and Credential Providers (CPs) can communicate effectively without custom integrations.

Think of it like HTTP for commerce - instead of every platform and business creating their own protocol, they can all speak UCP.

---

### What problem does UCP solve?

**Q: What specific pain points does UCP address?**

A: UCP solves several critical problems:

1. **Integration Complexity**: Without UCP, a platform supporting 100 businesses needs 100 custom integrations. With UCP, they need one implementation.

2. **Agentic Commerce**: AI agents need a standardized way to discover products, manage carts, and complete purchases on behalf of users - UCP is designed from the ground up for this.

3. **Security & Compliance**: UCP defines clear trust boundaries - raw payment data never touches platforms or businesses, only Credential Providers and PSPs handle sensitive information.

4. **Innovation Velocity**: Businesses can expose commerce capabilities once and be discovered by any UCP-compatible platform automatically.

---

### How is UCP different from existing payment standards?

**Q: We already have standards like Payment Request API, Apple Pay, etc. How is UCP different?**

A: Great question! UCP is complementary to, not competitive with, existing payment standards:

- **Payment Request API / Apple Pay / Google Pay**: Focus on payment instrument collection and tokenization (the CP role in UCP)
- **UCP**: Provides the full commerce lifecycle - product discovery, cart management, tax calculation, checkout orchestration, order management, and webhooks

UCP actually *uses* existing payment standards. For example, a UCP checkout flow might collect payment via Google Pay (which issues a token), then the business's PSP exchanges that token with Google to process the transaction.

UCP is higher-level - it's about the entire commerce interaction, not just payments.

---

## Architecture & Design

### Can you explain the "four actors" model?

**Q: What are the four actors in UCP and why this separation?**

A: UCP defines four distinct actors:

1. **Platform** - Consumer-facing surface (AI agents, apps, social media). Orchestrates checkout on behalf of users.

2. **Business** - Merchant of Record. Owns inventory, pricing, fulfillment, and order lifecycle.

3. **Credential Provider (CP)** - Manages sensitive user data (payment instruments, addresses). Issues tokens for privacy.

4. **Payment Service Provider (PSP)** - Processes actual transactions with card networks. Exchanges tokens with CPs.

**Why this separation?**

- **Separation of concerns**: Each actor focuses on their expertise
- **Security**: Raw payment data never touches Platform or Business
- **Flexibility**: Mix and match (any Platform + any Business + any PSP/CP)
- **Compliance**: Reduces PCI scope for Platform and Business

---

### What is the "server-selects" architecture?

**Q: I heard UCP uses "server-selects" instead of "client-selects". What does that mean?**

A: In UCP, the **business (server)** chooses which protocol version to use from the intersection of what both parties support.

**Traditional (client-selects)**: Client says "I want version X", server either supports it or returns 404.

**UCP (server-selects)**:
1. Client says "I support versions X, Y, Z"
2. Server says "I support versions Y, Z, W"
3. Server selects Y or Z (usually latest both support) and uses that

**Benefits**:
- Business controls rollout of new versions
- Easier A/B testing (business picks version per client)
- Simpler platform code (no complex fallback chains)
- Gradual evolution without breaking changes

---

### How does governance work without a central authority?

**Q: Who controls UCP? Is there a central registry?**

A: UCP uses **reverse-domain namespacing** - there's no central authority or registry.

- **Official UCP capabilities**: `dev.ucp.shopping.checkout`, `dev.ucp.common.identity_linking`
- **Custom vendor capabilities**: `com.shopify.payments.installments`, `com.example.loyalty.rewards`

The namespace format is: `{reverse-domain}.{service}.{capability}`

**Security binding**: The `spec` and `schema` URLs MUST have origins matching the namespace authority. This prevents capability hijacking:
- `dev.ucp.*` capabilities MUST reference specs at `https://ucp.dev/...`
- `com.shopify.*` capabilities MUST reference specs at `https://shopify.com/...`

Domain ownership proves authority - DNS is the governance mechanism.

---

## Technical Implementation

### How does discovery work?

**Q: How does a platform discover what a business supports?**

A: Every UCP-compliant business exposes a machine-readable profile at `/.well-known/ucp` (similar to `.well-known/openid-configuration`).

```javascript
// Platform discovers business capabilities
const profile = await fetch('https://shop.example.com/.well-known/ucp');
const data = await profile.json();

// Check what's supported
console.log(data.capabilities);
// => [{ name: "dev.ucp.shopping.checkout", version: "2026-01-11", ... }]

console.log(data.services.shopping.rest.endpoint);
// => "https://shop.example.com/api/ucp"
```

The profile declares:
- Supported capabilities and versions
- Transport bindings (REST, MCP, A2A)
- Payment handlers (Google Pay, Apple Pay, etc.)
- Signing keys for verification
- Webhook endpoints

Platforms cache profiles (recommended 1 hour) for performance.

---

### What transport protocols does UCP support?

**Q: Can I use UCP with REST APIs? GraphQL? gRPC?**

A: UCP is **transport-agnostic**. The initial focus is on three transports:

1. **REST** - Traditional HTTP APIs (OpenAPI specs)
2. **MCP (Model Context Protocol)** - For AI agents to call functions (OpenRPC specs)
3. **A2A (Agent-to-Agent)** - Future protocol for agent-native commerce

You declare your transport in the profile:

```json
{
  "services": {
    "shopping": {
      "rest": {
        "endpoint": "https://shop.example.com/api/ucp",
        "schema": "https://ucp.dev/services/shopping/rest.openapi.json"
      },
      "mcp": {
        "endpoint": "https://shop.example.com/mcp",
        "schema": "https://ucp.dev/services/shopping/mcp.openrpc.json"
      }
    }
  }
}
```

GraphQL or gRPC could be added as transports in the future - the data models remain the same.

---

### How does checkout work end-to-end?

**Q: Can you walk through a complete checkout flow?**

A: Here's a simplified flow:

```javascript
// 1. Platform discovers business capabilities
const profile = await fetch('https://shop.example.com/.well-known/ucp');

// 2. Platform creates checkout session
const checkout = await fetch('https://shop.example.com/api/ucp/checkout/create', {
  method: 'POST',
  body: JSON.stringify({
    items: [{ sku: 'ABC123', quantity: 1 }],
    buyer: { email: 'user@example.com' }
  })
});
// Business returns: { checkout_id, items, subtotal, tax, total }

// 3. User selects payment via Credential Provider (e.g., Google Pay)
const paymentToken = await googlePay.tokenize(cardDetails);
// CP returns token (raw card data never exposed)

// 4. Platform updates checkout with payment
await fetch('https://shop.example.com/api/ucp/checkout/update', {
  method: 'POST',
  body: JSON.stringify({
    checkout_id: checkout.checkout_id,
    payment: { token: paymentToken, provider: 'google_pay' }
  })
});

// 5. Business forwards to PSP, PSP exchanges token with CP
// (happens server-side at Business)

// 6. Platform completes checkout
const order = await fetch('https://shop.example.com/api/ucp/checkout/complete', {
  method: 'POST',
  body: JSON.stringify({ checkout_id: checkout.checkout_id })
});
// Business returns: { order_id, status: 'confirmed' }
```

**Key points**:
- Business calculates tax/shipping (they know their rules)
- CP tokenizes payment (security/privacy)
- PSP processes transaction (financial infrastructure)
- Platform orchestrates but doesn't touch sensitive data

---

### What are Extensions?

**Q: What's the difference between Capabilities and Extensions?**

A: **Capabilities** are core features that define a service:
- `dev.ucp.shopping.checkout` - Core checkout capability
- `dev.ucp.shopping.order` - Order lifecycle management

**Extensions** enhance capabilities without bloating them:
- `dev.ucp.shopping.discount` - Applies discounts to checkout
- `dev.ucp.shopping.fulfillment` - Adds shipping method selection
- `dev.ucp.shopping.gift_message` - Enables gift messages

Extensions are **optional** and **composable**:

```json
{
  "name": "dev.ucp.shopping.checkout",
  "extensions": [
    "dev.ucp.shopping.discount",
    "dev.ucp.shopping.fulfillment",
    "dev.ucp.shopping.gift_message"
  ]
}
```

Platform checks which extensions business supports and adapts UI accordingly.

---

## Security & Compliance

### How does UCP handle payment security?

**Q: How does UCP ensure payment data stays secure?**

A: UCP uses **tokenization** and **strict trust boundaries**:

1. **User provides payment to CP** (e.g., Google Pay) - raw card data never leaves CP
2. **CP issues token** - opaque string that's useless without CP
3. **Platform sends token to Business** - token has no intrinsic value
4. **Business forwards token to PSP** - PSP knows which CP issued it
5. **PSP exchanges token with CP** - only PSP can decrypt/use token

**Result**: Raw payment data NEVER touches Platform or Business.

Additional security:
- **Request signing**: Businesses can sign requests (JWS) to prove authenticity
- **Webhook signatures**: Verify webhooks come from legitimate business
- **AP2 mandates**: PSPs can require additional authentication (3DS) via standardized flow

---

### What about PCI compliance?

**Q: Do I need to be PCI compliant to implement UCP?**

A: **Platform**: No - you never touch raw payment data, only tokens.

**Business**: Depends on your PSP:
- If PSP handles tokens (recommended), you're likely **PCI-SAQ A** (minimal compliance)
- If you process raw cards, you need full **PCI-DSS** compliance (but why would you?)

**CP & PSP**: Yes - they handle raw payment data and must be PCI-DSS compliant.

UCP's architecture **minimizes compliance burden** for most participants.

---

## Use Cases & Adoption

### What are the main use cases?

**Q: Who should use UCP and for what?**

A: UCP is designed for:

**Platforms (AI Agents, Apps)**:
- AI shopping assistants (ChatGPT, Gemini) discovering and purchasing products
- Super apps (WeChat, Grab) integrating commerce from multiple businesses
- Voice assistants (Alexa, Google Assistant) completing purchases
- Social media (Instagram, TikTok) enabling in-feed checkout

**Businesses (Merchants)**:
- E-commerce platforms (Shopify, WooCommerce) exposing checkout APIs
- Vertical marketplaces (Airbnb, Uber) enabling agent-driven bookings
- SaaS providers selling subscriptions via agents
- Any business wanting to be discoverable by AI agents

**PSPs & CPs**:
- Payment processors integrating with UCP ecosystem
- Digital wallets supporting UCP checkout flows

---

### Can AI agents really shop autonomously?

**Q: How does UCP enable agentic commerce?**

A: UCP is designed for AI agents to act on behalf of users:

1. **Discovery**: Agent searches for "coffee maker under $100"
2. **Profile Fetch**: Agent finds business at `https://appliances.example.com`, fetches `/.well-known/ucp`
3. **Capability Check**: Agent sees business supports `dev.ucp.shopping.checkout`
4. **MCP Call**: Agent uses MCP transport to call `checkout.create(items=[{sku:"CM100", quantity:1}])`
5. **User Approval**: Agent presents checkout to user: "Found Breville Coffee Maker for $89.99, confirm purchase?"
6. **Payment**: Agent uses user's pre-approved payment method (tokenized by CP)
7. **Complete**: Agent calls `checkout.complete()` - order placed!

UCP provides the **standardized interface** agents need. The MCP transport is specifically designed for function calling by LLMs.

---

### What businesses should implement UCP first?

**Q: Should every business implement UCP immediately?**

A: Start with these criteria:

**Implement UCP if**:
- You want AI agents to discover and purchase from you
- You're building a platform/app that needs to integrate with multiple businesses
- You're in a fragmented vertical (travel, services) needing standardization

**Wait if**:
- You're a small business with low transaction volume
- Your commerce is highly specialized/custom (UCP works best for standardizable flows)
- You don't have development resources for API implementation

**Recommendation**: If you're already building a checkout API, make it UCP-compliant. The incremental effort is small, but the discoverability benefit is huge.

---

## Development & Integration

### How do I get started implementing UCP?

**Q: I want to make my business UCP-compliant. Where do I start?**

A: Follow this path:

1. **Read the spec**: https://ucp.dev/specification/overview
2. **Explore samples**: https://github.com/Universal-Commerce-Protocol/samples
3. **Use SDKs**:
   - TypeScript: `npm install @ucp/sdk-typescript`
   - Python: `pip install ucp-sdk`
4. **Implement in order**:
   - Create `/.well-known/ucp` profile
   - Implement `checkout.create` handler
   - Implement `checkout.update` handler
   - Implement `checkout.complete` handler
   - Add extensions (discount, fulfillment) as needed
5. **Test with conformance suite**: https://github.com/Universal-Commerce-Protocol/conformance
6. **Submit to UCP directory** (coming soon)

**Estimated effort**: 1-2 weeks for basic checkout capability (assuming existing commerce backend).

---

### Are there SDKs available?

**Q: What SDKs and tools exist for UCP?**

A: Current SDKs (check https://github.com/orgs/Universal-Commerce-Protocol for latest):

- **TypeScript/JavaScript**: Full client + server SDK
- **Python**: Server-side implementation (Pydantic models)
- **Conformance Tests**: Automated testing suite to verify compliance

**Coming soon**:
- Go SDK
- Java/Kotlin SDK
- Ruby SDK
- CLI tools for testing

**Documentation**:
- Full spec at https://ucp.dev
- Interactive learning materials in this repo (`ucp-learning/`)
- Sample implementations

---

### How do schemas work in UCP?

**Q: I noticed `source/` and `spec/` directories. What's the relationship?**

A: UCP uses a **source-to-spec generation model**:

- **Source schemas** (`source/`) contain annotated JSON schemas with UCP-specific metadata
- **Generated specs** (`spec/`) are auto-generated per-operation schemas (create_req, update_req, response variants)

**Annotations** control field behavior:
- `"ucp_request": "omit"` - field won't appear in create/update requests
- `"ucp_request": "optional"` - field is optional in requests
- `"ucp_response": "omit"` - field won't appear in responses

**Why?**: Different operations need different schemas. A `checkout_create_req.json` includes only fields needed to create a checkout, while `checkout_resp.json` includes generated fields like `checkout_id` and `created_at`.

**Golden rule**: Never edit `spec/` directly - always edit `source/` and regenerate.

---

### How do I test my UCP implementation?

**Q: How can I verify my implementation is correct?**

A: Testing options:

1. **Schema Validation**:
```bash
python validate_specs.py  # Validates all JSON/YAML
```

2. **Conformance Tests**:
```bash
# Clone conformance repo
git clone https://github.com/Universal-Commerce-Protocol/conformance
cd conformance
npm install

# Test your implementation
npm test -- --endpoint=https://your-shop.com/api/ucp
```

3. **Manual Testing**:
   - Use samples as reference implementations
   - Compare your responses against spec schemas
   - Test with UCP-compatible platforms (coming soon)

4. **Documentation Build**:
```bash
mkdocs serve  # Verify your extensions appear correctly in docs
```

---

## Comparison & Competition

### How does UCP compare to Web3/blockchain commerce?

**Q: Is UCP related to Web3, NFTs, or blockchain?**

A: No. UCP is a **traditional web protocol** using HTTP, JSON, and REST/RPC.

**UCP is**:
- Standard HTTP APIs
- JSON data formats
- OAuth 2.0 for identity
- JWS for signatures
- Works with existing payment rails (Visa, Mastercard, etc.)

**UCP is NOT**:
- Blockchain-based
- Using cryptocurrency
- Requiring wallets or private keys
- Decentralized in the Web3 sense

UCP is about **interoperability between existing commerce systems**, not creating new financial infrastructure.

---

### How does UCP relate to ActivityPub / Fediverse?

**Q: Could UCP be used with ActivityPub for decentralized commerce?**

A: Interesting idea! UCP and ActivityPub are complementary:

- **ActivityPub**: Decentralized social networking protocol
- **UCP**: Commerce interoperability protocol

You could imagine:
- Mastodon post with product link
- Link points to UCP-compliant business
- User's Mastodon app fetches `/.well-known/ucp`, enables in-app checkout
- Transaction flows through UCP

UCP doesn't specify *how* products are discovered (that's ActivityPub's job), it specifies *how* transactions happen once discovered.

---

### What about GraphQL Commerce extensions?

**Q: Should I use UCP or GraphQL Commerce?**

A: They serve different purposes:

**GraphQL Commerce** (like Shopify's Storefront API):
- Query language for commerce data
- Flexible data fetching
- Good for building custom storefronts
- Vendor-specific implementations

**UCP**:
- Standardized operation semantics across vendors
- Designed for agent/platform orchestration
- Discovery + execution protocol
- Vendor-neutral standard

You could **expose UCP via GraphQL** - UCP defines *what* operations mean, GraphQL defines *how* to call them.

---

## Roadmap & Future

### What's on the UCP roadmap?

**Q: What features are coming next?**

A: From the docs (https://ucp.dev/documentation/roadmap/):

**Near term**:
- **Loyalty** capability for reward programs
- **Personalization** signals for product discovery
- **Subscription** management
- More **sample implementations**

**Medium term**:
- **Travel** vertical (flights, hotels, car rentals)
- **Services** vertical (appointments, bookings)
- **Inventory** real-time stock checking
- **A2A transport** (agent-to-agent protocol)

**Long term**:
- **Multi-party transactions** (split payments, group purchases)
- **Returns & refunds** lifecycle
- **Analytics & reporting** standards

Community input shapes the roadmap - join discussions at https://github.com/Universal-Commerce-Protocol/ucp/discussions

---

### Will UCP support non-shopping verticals?

**Q: Can UCP be used for services, travel, or other industries?**

A: Yes! The initial release focuses on **Shopping** because it's the most common use case, but UCP is designed to support any vertical:

**Current**:
- `dev.ucp.shopping.*` - Physical and digital goods

**Planned**:
- `dev.ucp.travel.*` - Flights, hotels, car rentals
- `dev.ucp.services.*` - Appointments, bookings, consultations
- `dev.ucp.subscriptions.*` - Recurring services

The architecture is intentionally **composable**. Each vertical defines its own:
- Data models (e.g., `flight` vs `product`)
- Capabilities (e.g., `dev.ucp.travel.booking` vs `dev.ucp.shopping.checkout`)
- Extensions (e.g., `dev.ucp.travel.seat_selection`)

Vendors can also create vertical-specific capabilities under their namespace (e.g., `com.airbnb.travel.experiences`).

---

### How can I contribute to UCP?

**Q: I want to help shape UCP. How can I contribute?**

A: Multiple ways to contribute:

1. **GitHub Discussions**: Propose features, ask questions
   - https://github.com/Universal-Commerce-Protocol/ucp/discussions

2. **Issues**: Report bugs, suggest improvements
   - https://github.com/Universal-Commerce-Protocol/ucp/issues

3. **Pull Requests**: Contribute to spec, docs, or samples
   - See CONTRIBUTING.md for guidelines
   - Sign CLA (Contributor License Agreement)

4. **Implement & Share**: Build UCP-compliant systems and share learnings

5. **Spread the word**: Write blog posts, give talks, educate others

**Key principle**: UCP is **community-driven**. The spec evolves based on real-world implementation feedback.

---

### Is UCP open source?

**Q: Can anyone use UCP? What's the license?**

A: Yes, UCP is fully open source:

- **License**: Apache 2.0 (permissive, commercial-friendly)
- **No fees**: Free to implement, no licensing costs
- **No membership required**: Anyone can implement UCP
- **Spec is public**: https://ucp.dev
- **Code is public**: https://github.com/Universal-Commerce-Protocol

You can implement UCP in proprietary software, open source projects, or anywhere else without restrictions.

---

## Practical Concerns

### What about existing commerce APIs?

**Q: I already have a checkout API. Do I need to replace it with UCP?**

A: No! You have options:

1. **Adapter Layer**: Keep your existing API, add UCP adapter that translates
   ```
   UCP Request → Adapter → Your Existing API
   Your API Response → Adapter → UCP Response
   ```

2. **Parallel Implementation**: Support both your API and UCP
   ```
   /api/checkout/create  → Your existing API
   /api/ucp/checkout/create  → UCP API
   ```

3. **Gradual Migration**: Implement UCP for new features, migrate old features over time

Most businesses start with option 1 (adapter) since it's quickest to implement.

---

### Does UCP handle tax calculation?

**Q: How does UCP handle complex tax rules (sales tax, VAT, GST)?**

A: UCP **delegates tax calculation to the Business**:

1. Platform creates checkout with items + shipping address
2. **Business calculates tax** using their existing tax engine (TaxJar, Avalara, internal)
3. Business returns checkout with tax breakdown

**Why?**: Tax rules are incredibly complex and vary by:
- Business nexus (where they have presence)
- Product category (food vs clothing vs digital goods)
- Jurisdiction (state, county, city)
- Customer type (B2C vs B2B)

**Business knows their tax obligations** - UCP doesn't try to replicate that knowledge. Platform just displays the tax breakdown returned by Business.

---

### What about internationalization?

**Q: Does UCP support multiple currencies and languages?**

A: Yes:

**Currency**:
- Prices include `currency` field (ISO 4217 codes: USD, EUR, JPY)
- Business decides which currencies to support
- Platform can request specific currency (if business supports)

**Language**:
- Use standard HTTP `Accept-Language` header
- Business returns localized text (product names, descriptions) in response
- Errors include localizable error codes

**Example**:
```javascript
fetch('https://shop.example.com/api/ucp/checkout/create', {
  headers: {
    'Accept-Language': 'es-MX',  // Spanish (Mexico)
  },
  body: JSON.stringify({
    items: [{sku: 'ABC', quantity: 1}],
    currency: 'MXN'  // Request prices in Mexican Pesos
  })
});
```

---

### How do webhooks work?

**Q: How does the business notify platform about order updates?**

A: UCP supports webhooks for asynchronous events:

**Business profile declares webhook topics**:
```json
{
  "webhooks": {
    "topics": ["order.shipped", "order.delivered", "order.cancelled"],
    "endpoint": "https://platform.example.com/webhooks/ucp"
  }
}
```

**Business sends signed webhook**:
```javascript
// When order ships
await fetch('https://platform.example.com/webhooks/ucp', {
  method: 'POST',
  headers: {
    'X-UCP-Signature': computeJWSSignature(payload),
    'X-UCP-Topic': 'order.shipped'
  },
  body: JSON.stringify({
    order_id: '12345',
    tracking_number: '1Z999...',
    carrier: 'ups',
    shipped_at: '2026-01-15T10:30:00Z'
  })
});
```

**Platform verifies signature** using business's public key from `/.well-known/ucp`.

---

### Can I use UCP in production today?

**Q: Is UCP production-ready?**

A: UCP is in **active development**:

**Current state**:
- ✅ Spec is stable (v2026-01-11)
- ✅ Core capabilities defined (checkout, order, identity)
- ✅ SDKs available (TypeScript, Python)
- ⏳ Limited production implementations
- ⏳ Platform support growing

**Recommendation**:
- **Early adopters**: Yes, implement now to shape the standard
- **Large enterprises**: Wait for more production references
- **Greenfield projects**: Great time to build on UCP
- **Existing systems**: Start with proof-of-concept

UCP is designed to be **incrementally adoptable** - you can implement just checkout first, add extensions later.

---

## Miscellaneous

### Why "Universal Commerce Protocol" and not "Open Commerce Protocol"?

**Q: Curious about the naming choice?**

A: "Universal" emphasizes **interoperability across all parties** (platforms, businesses, PSPs, CPs), not just "open source" or "open spec" (which it also is).

The goal is **universal adoption** - anyone can implement, anyone can participate, and all implementations interoperate.

---

### Who created UCP?

**Q: What's the origin story of UCP?**

A: UCP emerged from recognition that **agentic commerce** (AI agents shopping on behalf of users) requires standardization. Without a common protocol, every platform would need custom integrations with every business - that doesn't scale.

The spec is developed openly at https://github.com/Universal-Commerce-Protocol with community input. It builds on decades of web standards (HTTP, OAuth, JWS) rather than reinventing fundamentals.

---

### Where can I learn more?

**Q: What resources are available for learning UCP?**

A: Multiple resources:

1. **Official Documentation**: https://ucp.dev
   - Core concepts
   - Full specification
   - API reference

2. **Interactive Learning**: Open `ucp-learning/ucp-learning-mindmap.html` in this repo
   - 11 major sections
   - 53 topics with code examples
   - Searchable, tracks progress

3. **Sample Implementations**: https://github.com/Universal-Commerce-Protocol/samples
   - Reference business implementation
   - Reference platform implementation
   - Working code examples

4. **GitHub Discussions**: https://github.com/Universal-Commerce-Protocol/ucp/discussions
   - Ask questions
   - Share experiences
   - Propose features

5. **This repo**:
   - Schema sources (`source/`)
   - Generated specs (`spec/`)
   - Documentation generation tools

---

## Quick Reference

### Key URLs

- **Official Site**: https://ucp.dev
- **Specification**: https://ucp.dev/specification/overview
- **GitHub**: https://github.com/Universal-Commerce-Protocol/ucp
- **Samples**: https://github.com/Universal-Commerce-Protocol/samples
- **Discussions**: https://github.com/Universal-Commerce-Protocol/ucp/discussions

### Core Concepts

- **Four Actors**: Platform, Business, CP, PSP
- **Discovery**: `/.well-known/ucp`
- **Governance**: Reverse-domain namespacing
- **Versioning**: Server-selects architecture
- **Transport**: REST, MCP, A2A

### Main Capabilities

- `dev.ucp.shopping.checkout` - Cart and checkout management
- `dev.ucp.shopping.order` - Order lifecycle webhooks
- `dev.ucp.common.identity_linking` - OAuth 2.0 user authorization
- `dev.ucp.payments.token_exchange` - PSP/CP token exchange

### Getting Started

```bash
# Explore documentation
open https://ucp.dev

# Clone samples
git clone https://github.com/Universal-Commerce-Protocol/samples

# Install SDK
npm install @ucp/sdk-typescript
# or
pip install ucp-sdk
```

---

**Last Updated**: January 2026 | **Version**: 2026-01-11

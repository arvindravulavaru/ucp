# UCP Visual Assets

This folder contains presentation-ready visualizations for explaining the Universal Commerce Protocol (UCP).

## 🌐 Interactive HTML Versions (Recommended)

All visualizations are now available as **interactive HTML pages** that render perfectly in any browser!

**Start here:** Open `index.html` to browse all visuals with descriptions and recommendations.

### Quick Access

- **[index.html](index.html)** - Navigation hub for all visuals
- **[eli5.html](eli5.html)** - Simple, friendly explanation for everyone
- **[quick-reference.html](quick-reference.html)** - One-page summary card
- **[protocol-flow.html](protocol-flow.html)** - Animated end-to-end flow
- **[architecture.html](architecture.html)** - Comprehensive architecture breakdown

**Advantages of HTML versions:**

- ✅ Universal compatibility (any browser)
- ✅ Smooth animations and transitions
- ✅ Responsive design (mobile-friendly)
- ✅ No rendering issues
- ✅ Easy to share (just send the file)
- ✅ Can be embedded in presentations

---

## 📦 SVG Versions (Legacy)

SVG versions are still available but may have rendering issues in some viewers. Use HTML versions for best results.

## Available Visualizations

### 1. ELI5 Explainer (`ucp-eli5.svg`)

**Purpose:** Explains UCP in simple, accessible terms for anyone - no technical background needed!

**What it shows:**
- The problem: Every shop speaks a different language
- The solution: UCP as a universal translator
- Real-world analogies (USB-C, power outlets, email)
- Who uses it: AI assistants, stores, payment apps, shopping apps
- What it does: Seamless integration behind the scenes

**Best for:**
- Executive summaries
- Non-technical stakeholders
- Marketing materials
- Social media
- Quick introductions
- "What is UCP?" elevator pitch

**Features:**
- Friendly, conversational tone
- Emoji-driven visuals
- Simple analogies everyone understands
- No technical jargon
- Colorful and approachable design

---

### 2. Protocol Flow Animation (`ucp-protocol-flow-animated.svg`)

**Purpose:** Demonstrates the end-to-end UCP transaction flow from discovery to order completion.

**What it shows:**
- Discovery phase (Platform → Business profile)
- Checkout session creation
- Payment handler negotiation
- Token acquisition from Credential Provider
- Payment completion via PSP
- Order confirmation

**9 Sequential Steps:**
1. Discovery: GET /.well-known/ucp
2. Profile Response (capabilities, services)
3. POST /checkout-sessions (line_items, buyer)
4. Checkout Response (totals, payment.handlers[])
5. Payment Token Request (Platform ↔ Credential Provider)
6. Encrypted Token / Network Token
7. POST /checkout-sessions/{id}/complete
8. Authorize / Capture (Business → PSP)
9. Order Confirmation (status: complete)

**Best for:**
- Executive presentations
- Developer onboarding
- Demo walkthroughs
- Conference talks
- Technical documentation

**Features:**
- Animated arrows showing message flow
- Color-coded actors (Platform, Business, Credential Provider, PSP)
- Step-by-step progression with timing
- Clear labels and descriptions

---

### 3. Architecture Infographic (`ucp-architecture-infographic.svg`)

**Purpose:** Provides a comprehensive view of UCP's modular architecture.

**What it shows:**
- **Core Capabilities Layer:** Checkout, Identity Linking, Order, Payment Token Exchange
- **Extensions Layer:** Fulfillment, Discounts, AP2 Mandates, Custom Extensions
- **Transport Layer:** REST, MCP, A2A, Embedded Protocol
- **Security Layer:** HTTPS, OAuth 2.0, PCI-DSS, JWS Signatures, Verifiable Credentials
- **Key Features:** Dynamic Discovery, Composable Architecture, Transport Agnostic
- **Governance Model:** Namespace Authority, Spec URL Binding, Version Negotiation

**Best for:**
- Architecture reviews
- System design discussions
- Integration planning
- Technical deep-dives
- Training materials

**Features:**
- Layered architecture visualization
- Color-coded sections for easy scanning
- Comprehensive feature breakdown
- Governance and security highlights

---

### 4. Quick Reference Card (`ucp-quick-reference.svg`)

**Purpose:** A one-page reference card summarizing all essential UCP information.

**What it shows:**
- Four core pillars (Discovery, Checkout, Payments, Security)
- Key benefits (Modular, Transport Agnostic, AI-Ready, Open Standard)
- Standard capabilities with their namespace IDs
- Transaction flow in 6 steps
- Clean, scannable layout

**Best for:**
- Conference handouts
- Developer quick reference
- Email signatures
- Documentation headers
- Social media posts
- Print materials

**Features:**
- Square format (1080×1080px) perfect for social media
- High information density
- Professional design
- Easy to scan and understand

---

## Usage Tips

### Viewing the HTML Pages

**In a Browser:**

```bash
# Open the navigation hub
open ucp-visuals/index.html

# Or open specific visuals directly
open ucp-visuals/eli5.html
open ucp-visuals/protocol-flow.html
```

The HTML pages work offline and don't require any web server. Just double-click to open!

### For Presentations

**For non-technical audiences:**

1. **Start with ELI5 Explainer** (`eli5.html`) - Simple, friendly introduction
2. **Follow with Quick Reference** (`quick-reference.html`) - Key points summary
3. **Show Architecture** (`architecture.html`) - Only if they want more details

**For technical audiences:**

1. **Start with Architecture** (`architecture.html`) - System overview
2. **Follow with Protocol Flow** (`protocol-flow.html`) - Implementation walkthrough
3. **Use Quick Reference** (`quick-reference.html`) - Summarize key points

**Presentation Integration:**

- **Screen share directly:** Just open the HTML file and present
- **Export to PDF:** Print the HTML page to PDF from your browser
- **Embed in slides:** Take screenshots or convert to images if needed

### Viewing the SVGs (Legacy)

**In a Browser:**
- Simply open the `.svg` files directly in any modern browser
- Full animations will play automatically
- Smooth scaling at any resolution

**In Presentations:**
- Import SVGs directly into PowerPoint, Keynote, or Google Slides
- Convert to PNG/PDF if needed: Use browser print-to-PDF or tools like Inkscape
- For web presentations, embed directly using `<img>` or `<object>` tags

**Command-line conversion (if needed):**
```bash
# Using Inkscape
inkscape ucp-protocol-flow-animated.svg --export-type=png --export-width=2400

# Using Chrome headless
google-chrome --headless --screenshot --window-size=1920,1080 ucp-protocol-flow-animated.svg
```

---

## Customization

These SVGs are designed to be editable:

1. Open in vector graphics editor (Inkscape, Adobe Illustrator, Figma)
2. Modify colors, text, or layout as needed
3. Adjust animation timing in CSS if desired
4. Export in your preferred format

---

## File Specifications

### HTML Files (Recommended)

| File | Purpose | Animations | Responsive |
|------|---------|-----------|-----------|
| `index.html` | Navigation hub | Yes | Yes |
| `eli5.html` | Simple explanation | Fade-in, slide-up | Yes |
| `protocol-flow.html` | End-to-end flow | Sequential reveal | Yes |
| `architecture.html` | System architecture | Layer-by-layer fade | Yes |
| `quick-reference.html` | One-page summary | Scale-in | Yes |

### SVG Files (Legacy)

| File | Dimensions | Colors | Animation |
|------|-----------|---------|-----------|
| `ucp-eli5.svg` | 1000×1400px | Purple gradient, friendly pastels | Yes (subtle bounce) |
| `ucp-protocol-flow-animated.svg` | 1200×900px | Purple gradient background, color-coded actors | Yes (CSS animations) |
| `ucp-architecture-infographic.svg` | 1400×1000px | Dark theme with vibrant accents | Yes (fade-in effects) |
| `ucp-quick-reference.svg` | 1080×1080px | Purple gradient, clean white cards | Yes (fade-in effects) |

---

## Color Palette

**Protocol Flow:**

- Platform: Blue/Purple (`#4F46E5` → `#7C3AED`)
- Business: Green (`#059669` → `#047857`)
- Credential Provider: Red (`#DC2626` → `#B91C1C`)
- PSP: Orange (`#EA580C` → `#C2410C`)

**Architecture:**

- Core Capabilities: Blue (`#3B82F6`)
- Extensions: Green (`#10B981`)
- Transport: Amber (`#F59E0B`)
- Security: Red (`#EF4444`)

---

## License

These visualizations are part of the UCP specification repository and are licensed under Apache 2.0.

```text
Copyright 2026 UCP Authors

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
```

---

## Feedback & Contributions

Found an issue or want to suggest improvements?

- Open an issue in the main UCP repository
- Suggest changes via pull request
- Contact the UCP maintainers

---

## Quick Reference

**Need a quick visual for:**

- "What is UCP in simple terms?" → Use **ELI5 Explainer**
- "How does UCP work?" → Use **Protocol Flow Animation**
- "What's the architecture?" → Use **Architecture Infographic**
- "What capabilities exist?" → Use **Architecture Infographic** or **Quick Reference Card**
- "How secure is it?" → Use **Architecture Infographic** (Security layer) or **Quick Reference Card**
- "Can it work with AI agents?" → Use **Architecture Infographic** (Show MCP/A2A transport + AP2 Mandates)
- "Quick one-pager summary?" → Use **Quick Reference Card**
- "Explaining to executives/non-tech?" → Use **ELI5 Explainer**

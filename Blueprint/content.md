# Brand, Menu & Product Content Specification

**Audience:** Customer UI, owner UI, and content agents.  
**Read with:** `frontend.md` and `api-contract.md`.

## 1. Content principles

- Restaurant brand comes first; platform branding stays invisible or very subtle in the guest experience.
- Use short, appetizing, factual descriptions. Do not invent ingredients, dietary suitability, sourcing claims, awards, or allergen guarantees.
- Prices, availability, taxes, and preparation status must come from the API.
- Use consistent naming and capitalization across menu, kitchen ticket, bill, and reports.
- Keep copy easy to understand for diners and busy restaurant staff in India. Start with English; keep all strings localization-ready.

## 2. Restaurant brand configuration

The owner can configure these fields. API responses return an approved public subset only.

| Field | Example | Rules |
|---|---|---|
| Restaurant display name | The Saffron Table | Required; plain text |
| Logo | Owner-uploaded logo | Validate type/size; use alt text |
| Cover/hero image | Food/ambience photo | Owner-provided/licensed asset |
| Primary/accent colors | Brand colors | Validate contrast; fall back to defaults |
| Short tagline | `Good food. Good company.` | Optional, max 80 chars |
| Description | `Regional favourites, prepared fresh.` | Optional, max 240 chars |
| Outlet display name | `Salt Lake` | Optional |
| Table label | `Table 08` | Public table identifier, not internal DB ID |
| Contact/social links | Owner-managed | Validate URLs; optional |
| Ordering notice | `Last orders at 10:30 PM` | Optional, owner-controlled |

Use the design tokens in `frontend.md`; theme values should be loaded as CSS variables, not injected as arbitrary CSS.

## 3. Menu content model

### Category
- `name`, `description` (optional), `sort_order`, `is_active`, image (optional).
- Example categories: `Recommended`, `Starters`, `Main Course`, `Breads`, `Rice & Biryani`, `Desserts`, `Beverages`.

### Item
- `name`, `short_description`, `full_description`, `image`, `base_price_paise`, `is_available`, `sort_order`.
- Configured labels: vegetarian/non-vegetarian/vegan, spice level, popular/chef's choice, and declared allergens. Only show labels supported by restaurant data.
- Never infer `gluten-free`, `nut-free`, `dairy-free`, or similar from the dish name.

### Options and add-ons
- Option group has a title, required/optional flag, min/max selection, and ordered options.
- Examples: portion (`Half`, `Full`), spice (`Mild`, `Medium`, `Hot`), add-ons (`Extra cheese`, `Boiled egg`).
- The backend validates selections and calculates their prices. UI must show when a choice is required.

## 4. Illustrative seed menu (demo only)

This data is fictional and for development screenshots/tests only. Replace it with restaurant-approved names, recipes, allergen declarations, photos, and prices before a live pilot.

| Category | Item | Short description | Illustrative price |
|---|---|---|---:|
| Starters | Paneer Tikka | Char-grilled paneer with a spiced marinade. | ₹229 |
| Starters | Chicken Tikka | Tandoor-cooked chicken with house spices. | ₹289 |
| Main Course | Dal Tadka | Yellow lentils tempered with spices. | ₹189 |
| Main Course | Butter Paneer | Paneer in a creamy tomato-based gravy. | ₹259 |
| Rice & Biryani | Chicken Biryani | Fragrant rice layered with chicken and spices. | ₹299 |
| Breads | Tandoori Roti | Whole-wheat flatbread from the tandoor. | ₹35 |
| Desserts | Gulab Jamun (2 pcs) | Milk-solid dumplings in cardamom syrup. | ₹89 |
| Beverages | Masala Chaas | Chilled, spiced buttermilk. | ₹69 |

Do not copy the illustrative menu into production without owner approval. The demo menu is not an allergen or nutrition statement.

## 5. Guest-facing copy library

### Primary navigation and actions
- `View menu`
- `Add`
- `Customize`
- `Add to cart`
- `View cart`
- `Place order`
- `Track order`
- `Call waiter`
- `View bill`
- `Pay now`
- `Try payment again`
- `Continue browsing`

### Table and session
- `You're ordering at Table {tableLabel}.`
- `Your table is ready for ordering.`
- `We couldn't verify this table QR code. Please ask a team member for help.`
- `Ordering is currently paused. Please call a team member.`
- `Your table already has an open bill. Ask staff if you need help starting a new session.`

### Dietary/allergy warning (must preserve meaning)
- Heading: `Dietary requirements & allergies`
- Helper: `Tell us about allergies or food preferences before placing your order.`
- Required safety notice: `Please confirm allergies with our team. Food may be prepared in a shared kitchen, so cross-contact may occur. We cannot guarantee an allergen-free meal.`
- Distinguish labels: `Allergy` and `Preference`.
- Submission acknowledgement: `We've added this note to your order. Please speak to a team member to confirm what can be prepared safely.`

### Waiter request
- `Your request has been sent.`
- `A team member has acknowledged your request.`
- `This request has been resolved.`
- `You already have a request waiting. Our team will be with you.`

### Order statuses
- `Order placed`
- `Accepted by kitchen`
- `Preparing your order`
- `Ready`
- `Served`
- `Cancelled`
- `Order updates are temporarily delayed. We're reconnecting.`

### Bill/payment
- `This bill includes all orders placed at your table during this session.`
- `Your payment is being confirmed. Do not pay again while this status is pending.`
- `Payment confirmed`
- `Payment could not be completed. Your bill is still open.`
- `Your receipt is ready.`

### Errors and empty states
- Empty menu: `The menu isn't available right now. Please ask our team for help.`
- Empty cart: `Your cart is empty. Explore the menu to add something.`
- Network: `Connection lost. Your last confirmed action is shown below. Please retry when connected.`
- Item unavailable: `This item is currently unavailable.`
- Order failure: `We couldn't confirm your order. Check your order list before trying again.`
- Empty waiter inbox: `No service requests waiting.`
- Empty kitchen: `No orders in this view.`
- Empty reports: `No sales data for this date range.`

## 6. Staff and owner copy

Use action-first labels: `Accept order`, `Start preparing`, `Mark ready`, `Acknowledge`, `Resolve`, `Mark item unavailable`, `Regenerate QR`, `Revoke QR`, `Export CSV`.

Use explicit metric labels:
- `Gross sales` — sales before the defined deductions; show the metric definition.
- `Net sales` — define exactly what is excluded; do not use this label without a documented formula.
- `Amount collected` — verified captured payments, net of recorded refunds if that is the chosen calculation.
- `Outstanding balance` — amount due on open table sessions.
- `Average order value` — state whether based on placed or paid orders.

## 7. Content governance

- Owner/authorized manager approves menu names, descriptions, prices, photos, ingredients, declared allergens, and dietary labels.
- Store structured values separately from display copy (e.g., item category and declared allergens are not inferred from description text).
- Sanitize text and validate uploaded media. Do not allow arbitrary HTML in restaurant content.
- Keep a fallback logo/color system and alt text.
- Record menu changes, price changes, availability changes, tax-setting changes, and QR rotations in audit logs.

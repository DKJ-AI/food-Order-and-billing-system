# Frontend UI/UX Specification

**Audience:** UI/UX, React, content, and frontend QA agents.  
**Read with:** `README.md`, `content.md`, `api-contract.md`; read `realtime.md` for live screens.

## 1. UX objective

Make ordering effortless for a first-time guest who scans a QR code while seated at a table. The customer interface is mobile-first, visual, fast, accessible, and branded. Staff interfaces optimize speed, glanceability, and error prevention—not decorative marketing.

## 2. App structure and route groups

One React application with route-level layouts. Use route guards for staff roles; never rely on hiding buttons as authorization.

| Route | Audience | Purpose |
|---|---|---|
| `/r/:restaurantSlug/t/:qrToken` | Guest | QR landing, branding and table session |
| `/r/:restaurantSlug/t/:qrToken/menu` | Guest | Menu browsing |
| `/guest/session/:sessionId/cart` | Guest | Cart and order confirmation |
| `/guest/session/:sessionId/orders` | Guest | Order history/status for this table session |
| `/guest/session/:sessionId/bill` | Guest | Consolidated bill and checkout |
| `/staff/login` | Staff | Staff login |
| `/kitchen` | Kitchen | KDS screen |
| `/waiter` | Waiter | Service-request inbox and table context |
| `/owner` | Owner/manager | Dashboard and management navigation |
| `/owner/menu`, `/owner/tables`, `/owner/branding`, `/owner/settings` | Owner/authorized manager | Setup and management |

The route scheme can be refined during scaffolding, but preserve the role separation and API contract.

## 3. Visual direction

### Customer experience: premium, warm, appetizing
- Editorial food photography with consistent aspect ratios and intentional cropping.
- Brand logo/name at top; compact table indicator such as `Table 08`.
- Strong typography hierarchy; readable body text; clear prices and item availability.
- Warm neutral canvas; restaurant-specific primary/accent colors; restrained use of color for status and alerts.
- Menu cards with image, item name, short description, dietary badge, price, and clear add action.
- Fixed bottom cart summary while cart has items; avoid covering content or device safe areas.
- Avoid clutter, excessive animation, tiny controls, and modal-heavy navigation.

### Staff experience: operational, high-contrast, glanceable
- Dense but readable information; minimal decoration.
- Strong status labels and elapsed-time visibility.
- Large touch targets suitable for a tablet mounted in a kitchen.
- Do not communicate urgent allergy information by color alone; use icon/text/label as well.
- Kitchen cards must not auto-disappear when a status changes. Keep a visible history/ready lane until staff action or configured retention.

## 4. Responsive behavior

| Screen | Layout rules |
|---|---|
| Guest phone 320–430 px | Single column, bottom cart bar, sticky category navigation where useful, 44×44 px minimum touch targets |
| Guest tablet 600–900 px | Two-column menu grid where image/content remains legible; cart can become side panel |
| Desktop 1024 px+ | Centered menu max-width; optional category rail and cart column |
| KDS tablet/desktop | 2–4 ticket columns based on width; prevent ticket card from becoming too narrow |
| Waiter phone/tablet | Request list first; filters as compact chips; one-tap acknowledge/resolve with confirmation where destructive |
| Owner desktop | Sidebar + content; responsive tables with mobile card fallback; filters remain visible |

Use CSS safe-area insets for sticky bars. Do not make horizontal scrolling the only way to reach an important action.

## 5. Customer screens and behaviors

### A. QR landing / session start
- Load restaurant logo, name, cover/brand color, outlet name if present, and table label.
- Resolve QR token server-side; show a friendly invalid/disabled QR screen if unavailable.
- Create or resume the active table session. Prevent duplicate sessions for one table unless a manager explicitly starts a new one.
- Primary CTA: `View menu`.
- Show service hours or ordering paused state if restaurant has disabled ordering.

### B. Menu
- Header: logo/name, table label, open/closed ordering indicator, cart icon/count, language selector only when localization is implemented.
- Category chips; search; featured items optional; menu list/grid.
- Item card: photo, name, short description, vegetarian/non-vegetarian/spice/allergen indicators where configured, price, availability, `Add` action.
- Item detail drawer/modal: larger photo, complete description, base price, required/optional option groups, quantity, item note, allergy disclosure.
- Unavailable items cannot be added; show `Currently unavailable`, not a broken button.
- Avoid rendering an allergen badge as a guarantee that food is free from that allergen.

### C. Cart and order placement
- List items, quantities, selected options, item notes, remove/edit controls.
- Show a separate dietary/allergy section. Distinguish `Allergy` from `Preference`.
- Explain: `Please tell our team about allergies. We will confirm what can be prepared safely. Shared-kitchen cross-contact may occur.`
- Recalculate totals on server before placement. Client totals are previews only.
- Disable duplicate taps while submitting; use an idempotency key; provide retry for network failures without creating duplicate orders.
- Confirm the table label and order summary before submission.

### D. Order tracking
- Show each order number, placed time, current status, item list, and last update.
- Status timeline: `Placed → Accepted → Preparing → Ready → Served` (cancelled handled separately).
- Clearly distinguish order status from bill/payment status.
- Live updates via WebSocket; reconnect and poll fallback. Show `Reconnecting…` when stale.

### E. Call waiter
- Accessible from menu, order, and bill screens.
- Reason choices: `Need assistance`, `Water`, `Cutlery/napkins`, `Question about an item`, `Request the bill`, `Other`.
- Optional short note; prevent repeated accidental spam with visible pending state and reasonable server rate limit.
- States: `Request sent`, `Staff acknowledged`, `Resolved`; allow a new request after resolution.

### F. Bill and payment
- Session-level consolidated bill across all non-cancelled orders.
- Show line items, quantities, options, subtotal, discounts if supported, configured tax lines, service charge if enabled, rounding, total, amount paid, and balance due.
- Clearly identify tax/charge labels; do not invent or hard-code statutory rates.
- Checkout creates a server-side payment intent/order. Show pending state until verified backend status arrives.
- On failed/abandoned payment, show retry and preserve the open bill. On success, show receipt/invoice number and paid time.
- Do not show `Paid` based solely on a gateway redirect query parameter.

## 6. Kitchen Display System (KDS)

- Full-screen friendly layout with station selector if multiple prep stations are configured.
- New tickets visually stand out and optionally play a sound after user interaction enables audio.
- Ticket header: order number, table label, time placed, elapsed time, priority/state.
- Body: item quantities, variants/add-ons, preparation notes, dietary/allergy alerts, and customer note.
- Actions: `Accept`, `Start preparing`, `Mark ready`; transitions must match API rules.
- Keep ready tickets visible until acknowledged/served/archived per configuration. Never auto-delete tickets after a timeout.
- Filters: New, Preparing, Ready, All; station/category filter if supported.
- Connection indicator, reconnect behavior, and last-event timestamp.
- Kitchen role must not see payment credentials or owner-only sales figures.

## 7. Waiter console

- Default view: new/pending service requests sorted oldest first, with table and request reason prominent.
- One-tap acknowledge; resolving requires a deliberate action and optional internal note.
- Show time since request, current owner/assignee if available, and state.
- Table view may show active orders and dietary alerts relevant to service, but not payment provider secrets or unnecessary guest data.
- Use sound/visual alert sparingly and provide mute control.

## 8. Owner dashboard and management

### Dashboard
- KPI cards: gross sales, net sales, paid orders, average order value, and open table sessions.
- Date range and outlet filter near the top. Default to the restaurant's local day.
- Charts: sales trend, category/item performance, hourly demand, payment mix.
- Tables: best-selling items, recent orders, refunds/cancellations.
- Every chart needs a useful empty state and a text/table equivalent for accessibility.
- Clearly label gross/net and whether figures include tax/service charge.

### Management pages
- Menu categories/items, variants/add-ons, item availability.
- Branding: logo, colors, cover image, restaurant name/description, contact/social links.
- Tables/QR: create, regenerate/revoke QR, print-friendly QR view, table labels.
- Staff and permissions.
- Tax/service-charge configuration with effective date and preview; display warning that owner should confirm local legal/tax treatment with their accountant.

## 9. Design tokens

Initial token names; actual values may be refined after a visual prototype. Keep them centralized and themeable per restaurant.

```css
:root {
  --color-brand-primary: #7A3E2D;
  --color-brand-accent: #D98B45;
  --color-page: #FAF7F2;
  --color-surface: #FFFFFF;
  --color-text: #24211F;
  --color-muted: #746E69;
  --color-border: #E8E0D8;
  --color-success: #287A50;
  --color-warning: #A65C10;
  --color-danger: #B42318;
  --radius-card: 16px;
  --radius-control: 10px;
  --shadow-card: 0 4px 18px rgba(34, 24, 18, .07);
}
```

Do not hard-code the sample colors into components; consume CSS variables/theme config. Minimum body font target 16 px; line height 1.45–1.6. Use a consistent spacing scale (4/8/12/16/24/32 px).

## 10. Component inventory

Shared primitives: `AppShell`, `BrandHeader`, `TablePill`, `Button`, `IconButton`, `Input`, `Badge`, `Card`, `Dialog/Drawer`, `Tabs`, `Toast`, `Skeleton`, `EmptyState`, `ErrorState`, `ConfirmDialog`, `Price`, `ConnectionStatus`.

Customer: `MenuCategoryNav`, `MenuItemCard`, `MenuItemDetail`, `OptionGroup`, `QuantityStepper`, `CartBar`, `CartLineItem`, `DietaryForm`, `OrderTimeline`, `WaiterCallDialog`, `BillSummary`, `PaymentStatus`.

Kitchen: `KitchenBoard`, `KitchenTicket`, `TicketItem`, `TicketStatusActions`, `ElapsedTime`, `StationFilter`.

Waiter: `ServiceRequestList`, `ServiceRequestCard`, `RequestStatusAction`, `TableContextPanel`.

Owner: `KpiCard`, `DateRangeFilter`, `SalesChart`, `TopItemsTable`, `ReportTable`, `MenuEditor`, `BrandingEditor`, `QrTableManager`.

## 11. Accessibility and UX quality gates

- Keyboard navigation and visible focus in all staff/owner forms.
- Correct labels, semantic headings, dialog focus management, and screen-reader status announcements.
- WCAG AA contrast target; status must include text/icon, not color alone.
- Touch targets at least 44×44 CSS px where possible.
- Respect reduced-motion preference; no essential information conveyed only by animation/sound.
- Display loading, empty, error, stale/offline, unauthorized, and success states.
- Preserve user input during recoverable network failures.
- Test at 320 px width and common tablet widths before calling a page complete.

## 12. Frontend agent rules

- Read `api-contract.md`; do not invent endpoint names or payload keys.
- Use generated or shared TypeScript API types where available. If unavailable, create a local typed adapter and flag the missing contract—do not fork the contract.
- Keep API calls in feature services/hooks, not scattered in page components.
- Keep route-level role checks for UX and enforce permissions on the backend too.
- Use fixtures only for isolated component development; remove fake data from production views.
- Every feature must include loading, empty, error, and success states.

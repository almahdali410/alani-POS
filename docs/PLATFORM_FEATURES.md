# Web, Android & iOS Feature Mapping

All three clients consume the same API and share the same permission model. The table
below marks what ships on each platform. **P1** = MVP/must-have at launch,
**P2** = fast-follow, **—** = not planned for that platform.

| Feature | Web | Android | iOS | Notes |
|---|---|---|---|---|
| POS checkout (cart, discounts, tax, split payment) | P1 | P1 | P1 | Core on all three |
| Barcode scanning | P1 (camera/USB scanner) | P1 (camera + Bluetooth scanner) | P1 (camera + Bluetooth scanner) | Web supports USB HID scanners as keyboard-wedge input |
| Offline sale + background sync | P2 (service worker, limited) | P1 | P1 | Offline is a **mobile-first** requirement; web gets best-effort offline in a later phase |
| Held orders / open tickets | P1 | P1 | P1 | |
| Shift & cash drawer management | P1 | P1 | P1 | Cash drawer hardware trigger is mobile/web-with-peripheral only |
| Receipt printing (Bluetooth/USB/network printers) | P1 (network/USB) | P1 (Bluetooth/network) | P1 (Bluetooth/network) | |
| Email receipt | P1 | P1 | P1 | |
| Refunds & returns | P1 | P1 | P1 | |
| Product & inventory management | P1 | P2 (view + quick edit) | P2 (view + quick edit) | Full catalog editing is web-first; mobile gets a lightweight editor |
| Stock transfer / stock count | P1 | P2 | P2 | |
| Employee management, roles & permissions | P1 | — | — | Back-office, web-only |
| Clock-in/out attendance | P2 | P1 | P1 | Attendance is naturally a mobile/on-device action |
| Sales analytics & reports | P1 | P2 (summary dashboard) | P2 (summary dashboard) | Full report builder/export is web-first |
| Customer/CRM management | P1 | P1 (lookup + create) | P1 (lookup + create) | Full profile editing web-first |
| Loyalty points at checkout | P1 | P1 | P1 | |
| Multi-store switching | P1 | P1 | P1 | |
| Subscription & billing management | P1 | P2 (read-only status) | P2 (read-only status) | Payment/checkout is web-first (broader gateway UI support) |
| Tenant/business settings | P1 | — | — | Back-office, web-only |
| Master Admin Panel | P1 | — | — | Web-only by design |
| Public website | P1 | — | — | Web-only |
| Push notifications (low stock, shift reminders, announcements) | P2 (browser push) | P1 | P1 | |
| Barcode label printing | P2 | — | — | |
| Integrations configuration | P1 | — | — | Web-only setup; runtime effects (e.g., printers) apply on mobile |

**Rationale:** mobile apps are optimized as **till devices** — fast, offline-capable,
hardware-integrated checkout and light day-to-day operations (clock-in, quick stock
lookup, customer lookup). Deep configuration (catalog authoring, employee/role setup,
report building, billing, CMS, Master Admin) lives on **Web**, which has the screen
real estate and doesn't need offline support. This mirrors how the offline-sync design
in [ARCHITECTURE.md](./ARCHITECTURE.md#offline-first-mobile-sync-strategy) is scoped —
mobile syncs POS-critical data; heavy admin data stays online-only.

## Native Hardware Integrations (Mobile)

| Hardware | Android | iOS |
|---|---|---|
| Barcode scanner (camera) | ✅ (CameraX + ML Kit) | ✅ (AVFoundation + Vision) |
| Bluetooth barcode scanner | ✅ | ✅ |
| Bluetooth/network receipt printer (ESC/POS) | ✅ | ✅ |
| Cash drawer (via printer kick-out or USB) | ✅ | ✅ (via supported printer only — iOS USB access is limited) |
| Card reader (future payment terminal integration) | ✅ (roadmap) | ✅ (roadmap) |

## Web-Specific Considerations
- Responsive design covers three targets: back-office desktop (primary), tablet POS
  (secondary till form factor), and mobile-web (account/status checks on the go).
- Progressive Web App (PWA) manifest + service worker for installability and
  best-effort offline caching of the POS screen, upgraded to full offline parity in a
  later phase if demand warrants it (see [ROADMAP.md](./ROADMAP.md)).

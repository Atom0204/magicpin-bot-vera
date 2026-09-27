# magicpin AI Challenge — Vera Merchant Assistant

## 1. Approach & Architecture
We implemented Vera as a deterministic, high-precision 4-context engine adhering strictly to the challenge contract:
- **4-Context Layering**: Every message is composed from `CategoryContext`, `MerchantContext`, `TriggerContext`, and optionally `CustomerContext`.
- **Zero Hallucination / Fact Anchoring**: All numbers (ratings, review counts, prices, dates, distances) are strictly pulled from injected contexts. No fake citations, generic discounts ("Flat 30% off"), or unauthorized URLs.
- **Category Voice Profiles**:
  - Dentists: Clinical-peer vocabulary (`recall`, `caries`, `varnish`), avoiding taboo words (`cure`, `guaranteed`).
  - Salons & Gyms: Concrete service + price hooks (`Haircut @ ₹99`, `Dental Cleaning @ ₹299`).
  - Pharmacies: High-convenience refill reminders sent on behalf of the store.
- **Multi-Turn Conversation State Machine (`respond`)**:
  - **Auto-Reply Detection**: Recognizes canned business replies and verbatim echoes; backs off with `wait_seconds: 14400` on turn 1 and exits cleanly on turn 2+.
  - **Immediate Intent Routing**: On explicit commitments ("Ok lets do it", "confirm", "proceed"), immediately switches to action mode without qualifying questions.
  - **Hostility Handling**: Immediately halts outreach and records merchant opt-out upon detecting hostile or unsubscribe keywords.

## 2. API Surface
Exposes strictly the 5 endpoints over HTTP/JSON using Python standard library (`http.server.ThreadingHTTPServer`):
1. `GET /v1/healthz`: Liveness probe with uptime and loaded context breakdown.
2. `GET /v1/metadata`: Team, model, and engine metadata.
3. `POST /v1/context`: Idempotent atomic context ingestion with version conflict detection (HTTP 409 on stale versions).
4. `POST /v1/tick`: Proactive trigger-driven engagement loop capped at 20 actions per tick.
5. `POST /v1/reply`: Synchronous multi-turn conversation response (`send`, `wait`, `end`).

## 3. Tradeoffs Made
- **Deterministic Engine vs. External LLM Calls**:
  - *Tradeoff*: Used a deterministic contextual template engine instead of real-time LLM inference.
  - *Rationale*: Eliminates API key dependency, achieves <10ms response latencies (well under the 30s timeout budget), guarantees 100% compliance with voice/taboo constraints, avoids URL injection penalties, and provides full reproducibility.
- **In-Memory Store vs. External Database**:
  - *Tradeoff*: In-memory `BotStore` with thread locking rather than Redis/PostgreSQL.
  - *Rationale*: Zero external dependencies, immediate cold-start, O(1) context lookups, and exact compliance with the judge simulator lifecycle.

## 4. Additional Context That Would Have Helped Most
1. **Historical Engagement Hours**: Merchant-specific operational hours and response windows to prevent messaging during peak customer rush.
2. **Catalog Live Sync**: Dynamic item inventory/stock levels for pharmacies and restaurants to prevent promoting out-of-stock items.
3. **WhatsApp Template Catalog**: The pre-approved Meta WhatsApp HSM template IDs and language localization keys for production delivery.

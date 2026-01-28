# Architecture Blueprint

## Non-Functional Targets
- **Latency:** <150ms for POS item search and cart updates.
- **Offline uptime:** 100% for checkout and returns at register level.
- **Availability:** 99.9%+ for cloud services.

## Domain Model (Core Entities)
- **Tenant** → **Store** → **Location** → **Register**
- **Product** → **Variant** → **SKU**
- **Inventory Ledger** (movement-based) with on-hand snapshots
- **Price List** (store/location/segment)
- **Order** (sale/return), **Payment**, **Discount**, **Tax**
- **Employee**, **Role**, **Permission**, **Audit Event**

## Data Flow Overview
1. **POS checkout** writes to local store with optimistic UI.
2. A **sync worker** batches and uploads operations to the API.
3. API persists the order and emits events via the outbox.
4. Inventory service updates stock and publishes inventory deltas.
5. POS receives deltas and refreshes local snapshots.

## Offline Sync Protocol
- **Operation log:** Each client write is appended with a monotonic client timestamp and vector clock.
- **Conflict resolution:**
  - Order edits: last-write-wins with audit trails.
  - Inventory: reconcile via ledger entries, not direct stock overwrites.
- **Idempotency:** All operations include a client-generated UUID.

## Performance Patterns
- **Local indexing** on SKU, barcode, and category.
- **Precomputed cart totals** for tax and discount tiers.
- **Server-side read models** for reports; avoid hot paths.

## Reliability
- **Outbox pattern** for event emission.
- **Retry + backoff** for sync.
- **Dead-letter queue** for unrecoverable operations.

## Security
- Token-based auth (short-lived access tokens + refresh).
- Device attestation for register devices.
- Fine-grained RBAC.

## API Outline
- `POST /pos/orders` (idempotent)
- `POST /pos/orders/{id}/refunds`
- `GET /catalog/deltas?since=`
- `GET /inventory/deltas?since=`

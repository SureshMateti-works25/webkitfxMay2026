# Functional technical specification — webkitfx / catalog (May 2026)

## 1. Purpose and scope

This document defines how the **Sarees / commerce** vertical aligns with the **webkitfx** JSON layout engine and a **normalized catalog** contract. It is written for implementers (backend, frontend, search, and ops) so that services, indices, and UIs stay consistent.

**Out of scope here:** wireframe-level UI copy; exact REST path naming (only behavioral requirements).

**Canonical example bundle:** `webkitfxv2_May26/apps/sarees/src/config/catalog-domain-model.example.json` (schema stamp `version: "2.2"`). That file is a **contract / seed**, not loaded by the app shell today.

---

## 2. Platform stack (reference)

| Layer | Responsibility |
|-------|----------------|
| `@webkitfxv2/core-engine` | Form definitions, validation, conditions (`visibleWhen` / `enabledWhen`), path helpers |
| `@webkitfxv2/react-renderer` | `JsonForm`, layout resolution, default widget registry |
| `apps/sarees` | E-commerce shell: auth roles (shopper/vendor), JSON-driven screens |

Domain rules and widget ids are registered in the **app**, not in the core packages.

---

## 3. Catalog bundle partitions

The example JSON splits data by **lifecycle and authority**. Production may use separate databases or namespaces per partition.

| Partition | Authority | Notes |
|-----------|------------|--------|
| `source` | OLTP-style truth | Authoritative rows for catalog, pricing, inventory, tax, fulfillment, visibility, collections, promotions (slim), presets |
| `derived` | Rebuildable | Listing signals (sellable, restock flags, ATP, etc.) materialized from source + movements + tenant windows |
| `analytics` | Read models / warehouse | Views, trends, rating aggregates — not source of truth for checkout |
| `engagement` | Bounded context | Reviews, product feedback, app feedback |
| `localization` | Locale content | Keyed strings (`key` + `locale` + `text`) referenced by `*L10nKey` fields |
| `promotionRuleSets` | Versioned logic | Referenced by `promotion.ruleSetId`; `version` here is **rule-set document version**, not the same as per-row optimistic `version` on `source` rows |

**Principle:** avoid duplicating the same fact in `source` and `derived` (e.g. do not copy `publishedAt` into derived listing signals; read from `source.products`).

---

## 4. Source — core entity groups

### 4.1 Identity and taxonomy

- **tenants** — multi-tenant root.
- **productTypes** — `code`, optional `defaultUnitOfMeasureCode`, localized name key.
- **categories** — tree via `parentId`; **productCategories** join products to categories with `isPrimary`, `sort`.

### 4.2 Product, variants, attributes

- **products** — `slug`, `status`, `defaultLocale`, `titleL10nKey`, GST hints, `variantAxisAttributeDefIds` (ordered PDP axes), lifecycle timestamps (`publishedAt`, `availableFrom` / `availableTo`).
- **skus** — sellable unit; **unit** object: `unitOfMeasureCode`, `saleIncrement`, `minOrderQuantity`, `allowFractionalQty`.
- **skuAttributeTuples** — (sku, attributeDef, attributeValue) facts.
- **attributeDefs** — `valueKind`, `labelL10nKey`, **variantAxisOrder** (1 = primary picker), **displayType** (UI hint: swatch, pill, dropdown, radio, text, range), **filterable**, **searchable**, **sortable**, **facetSort** (`definition_order` vs `value_sort_key`).
- **attributeValues** — `code`, `labelL10nKey`, **sortKey** (for sortable facets), **swatchHex**, **searchSynonyms** (when searchable).

### 4.3 Units of measure

- **unitsOfMeasure** — stable `code` (e.g. `EA`, `MTR`), ISO where applicable, `labelL10nKey`.
- Inventory quantities are expressed in the SKU’s **unit** unless a future **multi-UOM** table is introduced.

### 4.4 Media

- **mediaAssets** — storage metadata; **productMedia** / **skuMedia** link assets to product or SKU with `role` and `sortOrder`.

### 4.5 Commercial

- **priceLists**, **priceRows** — amount in minor units, currency, effective window.
- **taxCategories**, **taxRates** — HSN/SAC and jurisdiction bps.
- **fulfillmentProfiles**, **productFulfillment** — lead time, dispatch location, international flag.
- **channelVisibility** — per channel: browse/search/purchase flags at product or SKU scope.

### 4.6 Inventory

- **locations** — warehouse / store.
- **inventoryPositions** — on-hand, reserved, `updatedAt` (high churn).
- **inventoryMovements** — append-only ledger lines (`qtyDelta`, `reason`, `refType` / `refId`).

### 4.7 Merchandising and promotions

- **collections** — manual (or future algorithmic) membership with sort and pins.
- **promotions** — `code`, `ruleSetId`, `status`, schedule, `priority`, `activationPresetId` (indirection to operational gates).
- **promotionActivationPresets** — channels, region policy, segments, coupon requirement, inventory gate, redemption caps (where modeled).

**promotionRuleSets** (outside `source` in the example) hold **eligibility** predicates and **reward** shapes: percent off, fixed amount off, buy-X-get-Y, free shipping, with **stackPolicy** and **application** scope.

---

## 5. Audit, soft delete, and concurrency (v2.2)

Mutable **source** rows (products, SKUs, attribute defs/values, prices, collections, promotions, etc.) carry:

| Field | Semantics |
|-------|-----------|
| `createdAt` / `updatedAt` | UTC instants |
| `createdBy` / `updatedBy` | Principal id (`u_*` user or `svc_*` service account) |
| `deletedAt` | Soft delete; `null` if active. Default listings **exclude** non-null |
| `version` | **Integer** optimistic-lock counter; increment on each successful update |
| `revision` | Monotonic audit sequence (optional; may align with outbox / change log) |

**Bundle root** `version` (string, e.g. `"2.2"`) is the **example schema stamp**, not the row lock.

**API behavior (recommended):** clients send `If-Match` with the last known row `version`; server returns **412** on mismatch. **Append-only** tables (e.g. inventory movements) may only set `createdAt` / `createdBy`.

---

## 6. Search, facets, and PDP

1. **Facets:** include attribute defs with `filterable: true`; order facet values per `facetSort` and `attributeValues.sortKey` where applicable.
2. **Search:** for defs with `searchable: true`, index resolved label text and `searchSynonyms` for the product family / SKU.
3. **Sort options:** defs with `sortable: true` drive PLP sort keys using `sortKey` on values.
4. **PDP variant picker:** order axes by `products.variantAxisAttributeDefIds`; within an axis, use `variantAxisOrder` and `displayType` for control rendering.

---

## 7. Derived and engagement (functional behavior)

- **derived.productListingSignals** / **skuListingSignals** — precomputed booleans and timestamps (recently added, restocked, new media, ATP, sellable). Rebuilt from source + config windows + movements.
- **analytics.*** — time-series views, trend scores, rating histograms — feed ranking and badges; not used as legal tax or inventory truth.
- **engagement.*** — reviews and feedback workflows (`status`); separate moderation and retention policies.

---

## 8. Localization

- Product titles, attribute labels, unit labels, collection titles, fulfillment names use **stable keys** (`*L10nKey`) resolved via `localization.strings` for `(key, locale)`.
- Fallback: request locale → product `defaultLocale` (see `commerceDocs.resolveLocalizedTitle` pattern in the example).

---

## 9. Runtime status (webkitfxv2_May26)

| Item | Status |
|------|--------|
| JSON forms + `JsonForm` | Implemented in repo |
| `catalog-domain-model.example.json` | **Not** wired into `getShell` / runtime loader |
| Promotion redemption counters / coupon tables | Described in comments / docs; not full `source` tables in example |

**Next implementation steps (when productizing):** define a typed loader (or API client) that maps partitions to services; emit search index documents from `source` + `derived`; enforce `version` on write paths.

---

## 10. Revision history

| Date | Change |
|------|--------|
| 2026-05-10 | Initial TECHNICAL.md aligned with catalog example `2.2` and webkitfxv2_May26 README |

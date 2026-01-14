# Merlynx Design Document

**Date:** 2026-01-14
**Status:** Approved
**GitHub Issues:** #1-#36

## Overview

Merlynx is a Lightspeed eCom C-Series app that syncs shop data (products, categories, pages, variants) and provides embeddable widget components for enhanced storefront functionality.

### Core Features (MVP)

1. **Search Bar** - Configurable full-text search across products, categories, and pages
2. **Quick View Modal** - Product preview popup without leaving the page
3. **Pop-up Alerts** - Configurable marketing/announcement notifications

## Architecture

### Tech Stack

| Component | Technology |
|-----------|------------|
| Monorepo | Nx with apps/ + libs/ structure |
| API | NestJS with SWC |
| Dashboard | Next.js (App Router) |
| Widget | Preact + Vite |
| Database | PostgreSQL + Prisma |
| Auth | Lightspeed signature verification + JWT |
| Payments | Stripe subscriptions |
| Hosting | VPS + Coolify |
| CDN | Cloudflare (free tier) |

### Monorepo Structure

```
merlynx/
├── apps/
│   ├── api/                 # NestJS backend
│   └── dashboard/           # Next.js frontend
├── libs/
│   ├── shared/
│   │   └── types/           # Shared TypeScript types
│   ├── lightspeed-client/   # Lightspeed API wrapper
│   ├── database/            # Prisma schema + client
│   └── widget/              # Preact widget components
├── nx.json
├── package.json
└── tsconfig.base.json
```

## Database Schema

### Core Models

```prisma
model Shop {
  id                String   @id @default(cuid())
  lightspeedId      String   @unique
  clusterId         String              // eu1, us1, etc.
  language          String              // nl, en, de, etc.
  apiKey            String              // Encrypted at rest
  apiSecret         String              // Encrypted at rest
  name              String
  domain            String

  // Subscription
  stripeCustomerId  String?
  plan              Plan     @default(FREE)

  // Relations
  products          Product[]
  categories        Category[]
  pages             Page[]
  alerts            Alert[]
  widgetConfig      WidgetConfig?
  searchConfig      SearchConfig?

  // Sync tracking
  lastSyncAt        DateTime?
  syncStatus        SyncStatus @default(PENDING)

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
}

model Product {
  id                String   @id @default(cuid())
  lightspeedId      String
  shopId            String
  shop              Shop     @relation(fields: [shopId], references: [id])

  title             String
  description       String?
  handle            String
  image             String?
  price             Decimal
  compareAtPrice    Decimal?

  // Full-text search
  searchVector      Unsupported("tsvector")?

  variants          Variant[]
  categories        Category[]

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  @@unique([shopId, lightspeedId])
}

model Variant {
  id                String   @id @default(cuid())
  lightspeedId      String
  productId         String
  product           Product  @relation(fields: [productId], references: [id])

  title             String
  sku               String?
  price             Decimal
  inventory         Int      @default(0)

  @@unique([productId, lightspeedId])
}

model Category {
  id                String   @id @default(cuid())
  lightspeedId      String
  shopId            String
  shop              Shop     @relation(fields: [shopId], references: [id])

  title             String
  handle            String
  parentId          String?

  searchVector      Unsupported("tsvector")?

  products          Product[]

  @@unique([shopId, lightspeedId])
}

model Page {
  id                String   @id @default(cuid())
  lightspeedId      String
  shopId            String
  shop              Shop     @relation(fields: [shopId], references: [id])

  title             String
  handle            String
  content           String?

  searchVector      Unsupported("tsvector")?

  @@unique([shopId, lightspeedId])
}

model Alert {
  id                String   @id @default(cuid())
  shopId            String
  shop              Shop     @relation(fields: [shopId], references: [id])

  title             String
  message           String
  type              AlertType @default(INFO)
  position          Position  @default(BOTTOM_RIGHT)
  enabled           Boolean   @default(true)

  showOnPages       String[]  @default([])
  delaySeconds      Int       @default(0)
  dismissable       Boolean   @default(true)

  createdAt         DateTime @default(now())
}

model WidgetConfig {
  id                String   @id @default(cuid())
  shopId            String   @unique
  shop              Shop     @relation(fields: [shopId], references: [id])

  searchEnabled     Boolean  @default(true)
  quickViewEnabled  Boolean  @default(true)
  alertsEnabled     Boolean  @default(true)

  primaryColor      String   @default("#000000")
  borderRadius      Int      @default(8)

  searchPlaceholder String   @default("Search products...")
  searchPosition    String?
}

model SearchConfig {
  id                String   @id @default(cuid())
  shopId            String   @unique
  shop              Shop     @relation(fields: [shopId], references: [id])

  searchFields      SearchField[]

  includeVariations Boolean  @default(false)
  partialMatch      Boolean  @default(true)
}

model SearchField {
  id              String       @id @default(cuid())
  searchConfigId  String
  searchConfig    SearchConfig @relation(fields: [searchConfigId], references: [id])

  field           String
  enabled         Boolean      @default(true)
  priority        Int

  @@unique([searchConfigId, field])
}

enum Plan {
  FREE
  STARTER
  PRO
}

enum SyncStatus {
  PENDING
  SYNCING
  COMPLETED
  FAILED
}

enum AlertType {
  INFO
  WARNING
  SUCCESS
  PROMO
}

enum Position {
  TOP_LEFT
  TOP_RIGHT
  TOP_CENTER
  BOTTOM_LEFT
  BOTTOM_RIGHT
  BOTTOM_CENTER
}
```

### Row Level Security

PostgreSQL RLS enforces shop isolation at the database level:

```sql
ALTER TABLE "Product" ENABLE ROW LEVEL SECURITY;

CREATE POLICY shop_isolation ON "Product"
  USING ("shopId" = current_setting('app.current_shop_id')::text);
```

Usage in NestJS:

```typescript
async withShop<T>(shopId: string, fn: () => Promise<T>): Promise<T> {
  await this.$executeRaw`SELECT set_config('app.current_shop_id', ${shopId}, true)`;
  return fn();
}
```

## Lightspeed Integration

### Authentication

Lightspeed uses Basic HTTP Authentication:

```typescript
const CLUSTER_URLS = {
  'eu1': 'https://api.webshopapp.com',
  'us1': 'https://api.shoplightspeed.com',
};

const authHeader = 'Basic ' + Buffer.from(`${apiKey}:${apiSecret}`).toString('base64');
```

### App Installation Flow

1. User clicks "Install" in Lightspeed App Store
2. Lightspeed redirects to: `https://app.merlynx.com/install?cluster_id=eu1&language=nl&shop_id=123&signature=abc&timestamp=xxx&token=xyz`
3. App verifies signature (MD5 hash)
4. Stores shop credentials (encrypted)
5. Redirects to dashboard with JWT session

Signature verification:

```typescript
const sorted = ['language', 'shop_id', 'timestamp', 'token'].sort();
let str = sorted.map(k => `${k}=${params[k]}`).join('');
str += APP_SECRET;
const expected = md5(str);
return expected === signature;
```

### Rate Limits (Per Shop)

| Period | Limit |
|--------|-------|
| Per second | 20 requests |
| Per 5 minutes | 300 requests |
| Per hour | 3,000 requests |
| Per day | 12,000 requests |

### Sync Strategy

| Trigger | Action |
|---------|--------|
| App install | Full initial sync |
| Webhook received | Single item update |
| Cron (every 6 hours) | Full sync as safety net |

### Webhooks

Register on install for real-time updates:

```typescript
await client.post('webhooks', {
  isActive: true,
  itemGroup: 'products', // or 'variants', 'categories'
  itemAction: '*',
  address: `https://api.merlynx.com/webhooks/lightspeed/${shopId}`,
  format: 'json',
  language: shop.language,
});
```

## Search Implementation

PostgreSQL full-text search with configurable fields and priority:

```sql
-- Create search vectors
ALTER TABLE "Product" ADD COLUMN search_vector tsvector;
CREATE INDEX product_search_idx ON "Product" USING GIN(search_vector);

-- Auto-update trigger
CREATE TRIGGER product_search_update
  BEFORE INSERT OR UPDATE ON "Product"
  FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();
```

### Configurable Search Fields

| Field | Description |
|-------|-------------|
| tags | Product tags |
| title | Product title |
| shortDescription | Short description |
| variantTitle | Variant titles |
| brand | Brand name |
| longDescription | Full description |
| specifications | SKU, EAN, etc. |

Users can enable/disable fields and set priority order via the dashboard.

## Widget Architecture

### Embedding

Single script tag in Lightspeed theme:

```html
<script src="https://cdn.merlynx.com/widget.js?shop=abc123" defer></script>
```

### Initialization Flow

```typescript
(async function MerlynxWidget() {
  const shopId = getShopIdFromScript();
  const config = await fetchConfig(shopId); // with localStorage fallback

  if (config.searchEnabled) initSearchBar(config);
  if (config.quickViewEnabled) initQuickView(config);
  if (config.alertsEnabled) renderAlerts(config.alerts);
})();
```

### Resilience

- Widget script served via Cloudflare CDN
- Config cached in localStorage (5 min TTL)
- Graceful fallback if API unreachable (shop keeps working, widgets just don't appear)

## Infrastructure

### Deployment (VPS + Coolify)

| Service | Resources |
|---------|-----------|
| api | 1 CPU, 1GB RAM |
| dashboard | 0.5 CPU, 512MB RAM |
| postgres | 1 CPU, 2GB RAM |

### Domains

| Domain | Purpose |
|--------|---------|
| merlynx.com | Marketing/landing |
| app.merlynx.com | Dashboard |
| api.merlynx.com | NestJS API |
| cdn.merlynx.com | Widget script (Cloudflare) |

## Billing

Stripe flat monthly subscription:

| Plan | Features |
|------|----------|
| FREE | Search bar, 1 alert |
| STARTER | All components, 5 alerts |
| PRO | Unlimited alerts, priority support |

## API Modules

| Module | Responsibility |
|--------|----------------|
| Auth | Lightspeed install, JWT sessions |
| Shops | Shop CRUD, stats |
| Sync | Cron scheduler, data processors |
| Webhooks | Lightspeed real-time events |
| Search | Configurable full-text search |
| Widget | Config endpoint for embedded script |
| Alerts | Alert CRUD |
| Billing | Stripe integration |

## Dashboard Pages

| Page | Purpose |
|------|---------|
| /overview | Shop stats, sync status |
| /components | Toggle components on/off |
| /components/search | Search field configuration |
| /components/alerts | Alert management |
| /styling | Colors, borders |
| /install | Widget installation instructions |
| /billing | Subscription management |

## GitHub Issues

Implementation tracked across 5 epics:

- **#1** - Project Setup & Infrastructure (#2-#6)
- **#7** - Database & Core Libraries (#8-#11)
- **#12** - NestJS API (#13-#20)
- **#21** - Next.js Dashboard (#22-#30)
- **#31** - Preact Widget (#32-#36)

## Security Considerations

1. **API tokens encrypted at rest** using application-level encryption
2. **RLS enforces shop isolation** at database level
3. **Signature verification** for Lightspeed install requests
4. **JWT sessions** for dashboard authentication
5. **HTTPS only** for all endpoints
6. **Terms of Service** with liability limitations

## Future Enhancements (Post-MVP)

- Typo tolerance with `pg_trgm` extension
- Dutch language search support
- Search analytics
- Additional widget components (mega menu, filters, carousels)
- A/B testing for alerts
- Multi-language dashboard

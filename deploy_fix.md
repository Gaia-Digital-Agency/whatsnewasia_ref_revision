# Deployment Fixes for GCP

This document summarizes all code changes required to deploy the WhatsNewAsia application to Google Cloud Platform (Cloud Run).

## Backend Changes

### 1. Cloud SQL Support
**File:** `config/config.js`

Added support for Cloud SQL Unix socket connections:
```javascript
const dbHost = process.env.DATABASE_HOST || "127.0.0.1";
const isCloudSQL = dbHost.startsWith('/cloudsql/');

const common = {
  host: isCloudSQL ? undefined : dbHost,
  dialectOptions: {
    charset: "utf8mb4",
    ...(isCloudSQL && { socketPath: dbHost }),
  },
  // ...
};
```

**File:** `config/database.js`
- Added `dialectOptions` to Sequelize constructor

### 2. Database Auto-Sync on Startup
**File:** `app.js`

Changed from simple authentication to full sync:
```javascript
db.sequelize
  .authenticate()
  .then(() => {
    console.log("✅ Database connected");
    return db.sequelize.sync({ alter: true });
  })
  .then(() => console.log("✅ Database schema synced"))
```

### 3. Redis Graceful Fallback
**File:** `redisClient.js`

Redis is now optional. If `REDIS_HOST` is not set or connection fails, the app continues with a noop mock:
```javascript
const REDIS_ENABLED = process.env.REDIS_ENABLED !== 'false' && process.env.REDIS_HOST;

const noopRedis = {
  get: async () => null,
  set: async () => 'OK',
  del: async () => 0,
  keys: async () => [],
  setex: async () => 'OK',
  expire: async () => 0,
};

export default redis || noopRedis;
```

### 4. API-Only Mode (DISABLE_SSR)
**File:** `app.js`

Added option to run backend as API-only service without SSR:
```javascript
if (process.env.DISABLE_SSR === 'true') {
  return res.status(200).json({ message: 'API server running. Use /api endpoints.' });
}
```

### 5. Robots.txt Sitemap Fix
**File:** `app.js`

Fixed incomplete sitemap URL:
```javascript
// Before
Sitemap: ${process.env.SITEMAP_URL}

// After
Sitemap: ${process.env.SITEMAP_URL}/sitemap.xml
```

### 6. Date Format Fix (ISO 8601)
**File:** `src/services/article.service.js`

Changed date formatting from unparseable format to ISO 8601:
```javascript
// Before
"dd-MM-yyyy HH:mm:ss ZZZZ"

// After
"yyyy-MM-dd'T'HH:mm:ssZZ"
```

### 7. Sequelize CLI Config
**File:** `.sequelizerc`

Changed config path for CommonJS compatibility:
```javascript
config: path.resolve("config/config.cjs"),
```

### 8. Structured Logging
**File:** `src/helpers/logger.js` (new file)

Added a logger utility to replace `console.log` throughout the codebase for better log management in production.

---

## Frontend Changes

### 1. New Environment Variables
**File:** `.env.production`

```env
VITE_WHATSNEW_BACKEND_URL=https://whatsnewasia-backend-850916858221.asia-southeast2.run.app
VITE_SITE_URL=https://whatsnewasia.com
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

### 2. Image URL Configuration
**Files:**
- `src/hooks/useArticle.ts`
- `src/components/front/Image.tsx`
- `src/pages/Front/Templates/Single.tsx`
- `src/pages/Front/Templates/SingleEvent.tsx`
- `src/pages/Front/Templates/SingleJob.tsx`
- `src/pages/Front/Templates/SingleHousing.tsx`
- `src/pages/Front/Templates/Housing.tsx`
- `src/pages/Front/Templates/Events.tsx`

Added `IMAGE_URL` constant with fallback:
```typescript
const IMAGE_URL = import.meta.env.VITE_IMAGE_URL || API_URL
```

### 3. SEO Meta Tags (Open Graph & Twitter)
**Files:**
- `src/pages/Front/Templates/Home.tsx`
- `src/pages/Front/Templates/Single.tsx`
- `src/pages/Front/Templates/SingleEvent.tsx`
- `src/pages/Front/Templates/SingleJob.tsx`
- `src/pages/Front/Templates/SingleHousing.tsx`

Added canonical URLs, Open Graph, and Twitter Card meta tags:
```tsx
<link rel="canonical" href={`${SITE_URL}/...`} />
<meta property="og:type" content="article" />
<meta property="og:title" content="..." />
<meta property="og:description" content="..." />
<meta property="og:url" content="..." />
<meta property="og:image" content="..." />
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="..." />
<meta name="twitter:description" content="..." />
<meta name="twitter:image" content="..." />
```

### 4. CSS Import Fix for Bundling
**Files:**
- `src/main.tsx`
- `src/mainAdmin.tsx`
- `src/main.html`
- `src/mainAdmin.html`

Moved CSS imports from HTML to TypeScript for proper Vite bundling:
```typescript
// In main.tsx and mainAdmin.tsx
import "./index.css";
import "./font.css";
```

Removed from HTML files:
```html
<!-- Removed -->
<link rel="stylesheet" href="/src/index.css">
<link rel="preload" href="/src/font.css" ...>
```

### 5. Indonesia Redirect Disabled
**Files:**
- `src/pages/Front/PathResolver.tsx`
- `src/components/front/NavLocation.tsx`

Commented out/removed the automatic redirect to whatsnewindonesia.com for Indonesia routes.

### 6. SSR Safety - Optional Chaining
**Files:**
- `src/components/front/About.tsx`
- `src/components/front/MobileMenu.tsx`
- `src/components/front/NavLogo.tsx`
- `src/pages/Front/Templates/Single.tsx`
- `src/pages/Front/Templates/SingleEvent.tsx`
- `src/pages/Front/Templates/Header.tsx`

Added optional chaining (`?.`) to prevent errors when SSR data is not available:
```typescript
// Before
initialData.article

// After
initialData?.article
```

### 7. PathResolver Client-Side Fix
**File:** `src/pages/Front/PathResolver.tsx`

Added handling for `LOADING` state and trigger initial route resolution when no SSR data:
```typescript
if (routeType === 'LOADING' || !routeType) {
  setClientChange(true)
}

case "LOADING":
  return null
```

### 8. Sidebar Navigation Config Extracted
**File:** `src/config/sidebarNav.config.ts` (new file)

Extracted navigation items from `AppSidebar.tsx` into a separate config file for better maintainability.

---

## Required Environment Variables

### Backend (.env)
```env
DATABASE_HOST=/cloudsql/gda-p01:asia-southeast2:whatsnewasia-db  # or IP for non-Cloud SQL
DATABASE_USER=<username>
DATABASE_PASSWORD=<password>
DATABASE_NAME=whatsnewasia
REDIS_HOST=  # Optional, leave empty to disable
REDIS_PORT=6379
SITEMAP_URL=https://whatsnewasia.com
DISABLE_SSR=false  # Set to true for API-only mode
```

### Frontend (.env.production)
```env
VITE_WHATSNEW_BACKEND_URL=https://whatsnewasia-backend-850916858221.asia-southeast2.run.app
VITE_SITE_URL=https://whatsnewasia.com
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

---

## Deployment Commands

```bash
# Backend
gcloud builds submit ./backend/whatsnewasia_be_revision \
  --tag asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/backend \
  --region=asia-southeast2

gcloud run deploy whatsnewasia-backend \
  --image asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/backend \
  --region asia-southeast2

# Frontend
gcloud builds submit ./frontend/whatsnewasia_fe_revision \
  --tag asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region=asia-southeast2

gcloud run deploy whatsnewasia-frontend \
  --image asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region asia-southeast2
```

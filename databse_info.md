# WhatsnewAsia Database & Storage Configuration

## Overview

This document contains all database, image storage, and authentication configuration details for the WhatsnewAsia application deployed on GCP.

---

## Live Site URLs

### Production URLs

| Service | URL |
|---------|-----|
| **Frontend** | https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app |
| **Backend API** | https://whatsnewasia-backend-850916858221.asia-southeast2.run.app |
| **Custom Domain** | https://whatsnewasia.com |

### Development URLs

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5173 (Vite dev server) |
| Backend API | http://localhost:7777 |

### Code Locations for URL Configuration

| File | Line(s) | Purpose |
|------|---------|---------|
| `whatsnewasia_fe_revision/.env.production` | 1 | Production backend URL (`VITE_WHATSNEW_BACKEND_URL`) |
| `whatsnewasia_fe_revision/.env.production` | 2 | Production site URL (`VITE_SITE_URL`) |
| `whatsnewasia_fe_revision/.env` | 2-3 | Development URLs |
| `whatsnewasia_be_revision/.env` | 29 | Frontend URL whitelist (`FRONTEND_URL`) |

---

## GCP Resources (asia-southeast2)

| Resource | Name | Details |
|----------|------|---------|
| Cloud Run (Frontend) | `whatsnewasia-frontend` | `https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app` |
| Cloud Run (Backend) | `whatsnewasia-backend` | `https://whatsnewasia-backend-850916858221.asia-southeast2.run.app` |
| Cloud SQL | `whatsnewasia-db-asia` | MySQL 8.0 |
| Artifact Registry | `whatsnewasia-repo` | Docker images |
| Cloud Storage | `gda_p01_storage` | Image storage bucket |

---

## Database Configuration

### Credentials

| Setting | Value |
|---------|-------|
| Database User | `root` |
| Database Password | `admin123` |
| Database Port | `3306` |
| Dialect | MySQL 8.0 |
| Charset | `utf8mb4` |
| Collation | `utf8mb4_unicode_ci` |
| Timezone | `+07:00` |

### Connection Pool Settings

| Setting | Value | Environment Variable |
|---------|-------|---------------------|
| Max Connections | `10` | `DB_POOL_MAX` |
| Min Connections | `0` | `DB_POOL_MIN` |
| Idle Timeout | `10000` ms | `DB_POOL_IDLE` |
| Acquire Timeout | `30000` ms | `DB_POOL_ACQUIRE` |

### Code Locations for Database Configuration

| File | Line(s) | Purpose |
|------|---------|---------|
| `whatsnewasia_be_revision/config/config.js` | 5-6 | Cloud SQL detection logic |
| `whatsnewasia_be_revision/config/config.js` | 8-29 | Main Sequelize configuration |
| `whatsnewasia_be_revision/config/config.js` | 23 | Cloud SQL socket path setting |
| `whatsnewasia_be_revision/config/config.cjs` | 1-37 | CommonJS config for migrations |
| `whatsnewasia_be_revision/config/database.js` | 1-23 | Sequelize initialization |
| `whatsnewasia_be_revision/.env` | 5-12 | Database environment variables |

### Cloud SQL Connection

The application detects Cloud SQL by checking if `DATABASE_HOST` starts with `/cloudsql/`:

```javascript
// config/config.js:5-6
const dbHost = process.env.DATABASE_HOST || "127.0.0.1";
const isCloudSQL = dbHost.startsWith('/cloudsql/');
```

**For Cloud SQL deployment**, set `DATABASE_HOST` to the socket path:
```
DATABASE_HOST=/cloudsql/PROJECT_ID:asia-southeast2:whatsnewasia-db-asia
```

**For local development**, use standard host:
```
DATABASE_HOST=localhost
```

---

## Image Storage Configuration

Images are stored in **Google Cloud Storage**, NOT in Cloud SQL.

### GCS Bucket Details

| Setting | Value |
|---------|-------|
| Bucket Name | `gda_p01_storage` |
| Image Folder | `gda_wna_images` |
| Full URL | `https://storage.googleapis.com/gda_p01_storage/gda_wna_images` |
| GCP Console | `https://console.cloud.google.com/storage/browser/gda_p01_storage/gda_wna_images` |

### Code Locations for Image Configuration

| File | Line(s) | Purpose |
|------|---------|---------|
| `whatsnewasia_fe_revision/.env.production` | 3 | Production image URL (`VITE_IMAGE_URL`) |
| `whatsnewasia_fe_revision/.env` | 4 | Development image URL |
| `whatsnewasia_be_revision/app.js` | 338 | Backend image URL via `process.env.IMAGE_URL` |
| `whatsnewasia_be_revision/src/config/image.config.js` | 1-52 | Image compression presets |
| `whatsnewasia_be_revision/src/routers/asset_media.router.js` | 13-14 | Local upload directory |

### Environment Variables

**Frontend Production** (`whatsnewasia_fe_revision/.env.production`):
```
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

**Frontend Development** (`whatsnewasia_fe_revision/.env`):
```
VITE_IMAGE_URL=http://localhost:7777
```

**Backend** (`whatsnewasia_be_revision/.env`):
```
IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

### To Change Image Storage Location

1. Update `VITE_IMAGE_URL` in `whatsnewasia_fe_revision/.env.production`
2. Update `IMAGE_URL` in backend environment variables
3. Ensure the new storage location has proper CORS and public access configured

---

## Authentication Secrets

| Secret | Value | Environment Variable |
|--------|-------|---------------------|
| JWT Secret | `iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=` | `JWT_SECRET` |
| JWT Refresh Secret | `r8pxbqQu6PEsEc3D28EJJdWWhap2mRL2sUuV+AVq4Rk=` | `JWT_REFRESH_SECRET` |

---

## Backend Environment Variables Reference

Complete list of backend environment variables (`whatsnewasia_be_revision/.env`):

```env
# Server
PORT=7777
URL=http://localhost
NODE_ENV=development

# Database Connection
DATABASE_HOST=localhost                    # Use /cloudsql/... for Cloud SQL
DATABASE_PORT=3306
DATABASE_USER=root
DATABASE_PASSWORD=admin123

# Database Names
DATABASE=whatsnewasia                      # Development database
DATABASE_PRODUCTION=whatsnewasia           # Production database
DATABASE_TEST=whatsnewasia_test            # Test database

# Connection Pool
DB_POOL_MAX=10
DB_POOL_MIN=0
DB_POOL_IDLE=10000
DB_POOL_ACQUIRE=30000
DB_TIMEZONE="+07:00"

# Authentication
JWT_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=
JWT_REFRESH_SECRET=r8pxbqQu6PEsEc3D28EJJdWWhap2mRL2sUuV+AVq4Rk=

# CORS
WHITELIST_CORS_ORIGIN=[http://localhost:3000,http://localhost:7777]
FRONTEND_URL=http://localhost:5173,http://localhost:3000,http://localhost:5175

# Redis
REDIS_PORT=6379
REDIS_HOST=127.0.0.1

# Image Storage
IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

---

## Frontend Environment Variables Reference

**Development** (`whatsnewasia_fe_revision/.env`):
```env
VITE_WHATSNEW_BACKEND_URL=http://localhost:7777
VITE_SITE_URL=http://localhost:7777
VITE_IMAGE_URL=http://localhost:7777
```

**Production** (`whatsnewasia_fe_revision/.env.production`):
```env
VITE_WHATSNEW_BACKEND_URL=https://whatsnewasia-backend-850916858221.asia-southeast2.run.app
VITE_SITE_URL=https://whatsnewasia.com
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

---

## Quick Reference: File Locations

### Database Configuration Files
- `whatsnewasia_be_revision/config/config.js` - Main Sequelize config (ESM)
- `whatsnewasia_be_revision/config/config.cjs` - Migration config (CommonJS)
- `whatsnewasia_be_revision/config/database.js` - Database initialization
- `whatsnewasia_be_revision/.env` - Environment variables

### Image Configuration Files
- `whatsnewasia_fe_revision/.env.production` - Production image URL
- `whatsnewasia_fe_revision/.env` - Development image URL
- `whatsnewasia_be_revision/src/config/image.config.js` - Image compression settings
- `whatsnewasia_be_revision/src/routers/asset_media.router.js` - Upload handling

### URL Configuration Files
- `whatsnewasia_fe_revision/.env.production` - Production frontend/backend URLs
- `whatsnewasia_fe_revision/.env` - Development URLs
- `whatsnewasia_be_revision/.env` - Backend CORS and frontend whitelist

---

## Useful Commands

### Connect to Cloud SQL via Proxy
```bash
cloud_sql_proxy -instances=PROJECT_ID:asia-southeast2:whatsnewasia-db-asia=tcp:3306
```

### Run Database Migrations
```bash
cd whatsnewasia_be_revision
npm run db:migrate
```

### Run Database Seeds
```bash
cd whatsnewasia_be_revision
npm run db:seed
```

---

## Cloud Run Production Environment Variables

### Backend (whatsnewasia-backend)

These environment variables must be set in Cloud Run for the backend to work:

```env
# Required
NODE_ENV=production
PORT=8080

# Database (Cloud SQL)
DATABASE_HOST=/cloudsql/PROJECT_ID:asia-southeast2:whatsnewasia-db-asia
DATABASE_USER=root
DATABASE_PASSWORD=admin123
DATABASE_PRODUCTION=whatsnewasia

# CORS - IMPORTANT: Must include frontend URLs
FRONTEND_URL=https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app,https://whatsnewasia.com

# Authentication
JWT_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=
JWT_REFRESH_SECRET=r8pxbqQu6PEsEc3D28EJJdWWhap2mRL2sUuV+AVq4Rk=

# Image Storage
IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

### Update Backend CORS via gcloud

```bash
gcloud run services update whatsnewasia-backend \
  --region asia-southeast2 \
  --set-env-vars "FRONTEND_URL=https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app,https://whatsnewasia.com"
```

### CORS Configuration Location

| File | Line(s) | Purpose |
|------|---------|---------|
| `whatsnewasia_be_revision/app.js` | 76-82 | CORS allowed origins from `FRONTEND_URL` |
| `whatsnewasia_be_revision/app.js` | 84-95 | CORS middleware configuration |

---

## Notes

- Always use environment variables for sensitive data in production
- The application automatically detects Cloud SQL connections via the `/cloudsql/` prefix
- Image uploads in development are stored locally in `/uploads` directory
- Production images should be uploaded to GCS bucket directly or via admin interface
- **CORS**: Backend must have `FRONTEND_URL` set to allow frontend origins

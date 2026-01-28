# WhatsNewAsia

A full-stack content management and publishing platform for news and articles across Asian countries. The platform features a public-facing website for readers and an admin CMS for content creators and administrators.

## Live Application

| Environment | URL |
|-------------|-----|
| **Frontend (Public Site)** | https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app |
| **Admin Panel** | https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app/admin |
| **Backend API** | https://whatsnewasia-backend-850916858221.asia-southeast2.run.app/api |

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [GCP Services](#gcp-services)
- [Project Structure](#project-structure)
- [Frontend Structure](#frontend-structure)
- [Admin Panel](#admin-panel)
- [Authentication](#authentication)
- [Database Schema](#database-schema)
- [Local Development](#local-development)
- [Deployment](#deployment)
- [When to Redeploy](#when-to-redeploy)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)

## Project Overview

WhatsNewAsia is a multi-country content platform that allows:

- **Readers** to browse articles by country, city, and category
- **Content creators** to write, edit, and publish articles
- **Administrators** to manage users, categories, tags, locations, and site templates

### Key Features

- Multi-location content (countries, cities, regions)
- Category-based content organization (Holiday, Job Listing, Housing, Technology, etc.)
- Article versioning and draft management
- Scheduled publishing
- Trending articles algorithm
- Newsletter subscription
- Media library management
- Dynamic page templates
- Server-side rendering (SSR) for SEO

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Google Cloud Platform                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────┐         ┌─────────────────┐                   │
│   │   Cloud Run     │         │   Cloud Run     │                   │
│   │   (Frontend)    │◄───────►│   (Backend)     │                   │
│   │   Port: 8080    │         │   Port: 7777    │                   │
│   │   nginx + SSR   │         │   Express.js    │                   │
│   └────────┬────────┘         └────────┬────────┘                   │
│            │                           │                            │
│            │                           ▼                            │
│            │                  ┌─────────────────┐                   │
│            │                  │   Cloud SQL     │                   │
│            │                  │   (MySQL 8)     │                   │
│            │                  │   Instance:     │                   │
│            │                  │   gda-wna-mysql8│                   │
│            │                  └─────────────────┘                   │
│            │                                                        │
│            ▼                                                        │
│   ┌─────────────────┐                                               │
│   │  Cloud Storage  │                                               │
│   │  (GCS Bucket)   │                                               │
│   │  gda_p01_storage│                                               │
│   │  /gda_wna_images│                                               │
│   └─────────────────┘                                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Request Flow

1. **Public Site**: User requests → Cloud Run Frontend (nginx) → SSR renders page → API calls to Backend
2. **Admin Panel**: User requests → Cloud Run Frontend → React SPA → API calls to Backend
3. **API Requests**: Backend → Cloud SQL (via Unix socket) → Response
4. **Images**: Frontend constructs URLs pointing to GCS public bucket

## Tech Stack

### Backend (`whatsnewasia_be_revision`)

| Technology | Purpose |
|------------|---------|
| Node.js | Runtime environment |
| Express.js | Web framework |
| Sequelize | ORM for MySQL |
| MySQL 8 | Database |
| JWT | Authentication tokens |
| bcrypt | Password hashing |
| Multer | File uploads |
| Sharp | Image processing |
| ioredis | Redis client (caching) |
| Nodemailer | Email sending |
| Luxon | Date/time handling |

### Frontend (`whatsnewasia_fe_revision`)

| Technology | Purpose |
|------------|---------|
| React 19 | UI framework |
| TypeScript | Type safety |
| Vite | Build tool |
| React Router 7 | Routing |
| Tailwind CSS 4 | Styling |
| Axios | HTTP client |
| Quill | Rich text editor |
| ApexCharts | Dashboard charts |
| SweetAlert2 | Modals/alerts |
| Swiper | Carousels |
| react-dropzone | File uploads |

### Infrastructure

| Service | Purpose |
|---------|---------|
| Cloud Run | Container hosting |
| Cloud SQL | Managed MySQL |
| Cloud Storage | Image/asset storage |
| Cloud Build | CI/CD builds |
| Artifact Registry | Docker image storage |

## GCP Services

### Project Details

| Property | Value |
|----------|-------|
| Project ID | `gda-p01` |
| Region | `asia-southeast2` (Jakarta) |
| Backend Service | `whatsnewasia-backend` |
| Frontend Service | `whatsnewasia-frontend` |
| Cloud SQL Instance | `gda-wna-mysql8` |
| GCS Bucket | `gda_p01_storage/gda_wna_images` |

### Service URLs

| Service | URL |
|---------|-----|
| Frontend | https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app |
| Backend API | https://whatsnewasia-backend-850916858221.asia-southeast2.run.app/api |
| Image CDN | https://storage.googleapis.com/gda_p01_storage/gda_wna_images |

### Cloud SQL Connection

The backend connects to Cloud SQL via Unix socket when running in Cloud Run:
```
/cloudsql/godaexplore:asia-southeast2:gda-wna-mysql8
```

For local development, use Cloud SQL Proxy:
```bash
cloud-sql-proxy godaexplore:asia-southeast2:gda-wna-mysql8 --port=3306
```

---

## Project Structure

```
whatsnewasia/
├── whatsnewasia_be_revision/     # Backend (Node.js/Express)
│   ├── app.js                    # Main entry point
│   ├── Dockerfile                # Container definition
│   ├── package.json              # Dependencies
│   ├── config/
│   │   └── config.cjs            # Sequelize config
│   └── src/
│       ├── config/               # App configuration
│       ├── controllers/          # Route handlers
│       ├── helpers/              # Utility functions
│       ├── middlewares/          # Express middlewares
│       ├── migrations/           # Database migrations
│       ├── models/               # Sequelize models
│       ├── routers/              # API routes
│       ├── seeders/              # Database seeders
│       ├── services/             # Business logic
│       └── ssr/                  # Server-side rendering
│
├── whatsnewasia_fe_revision/     # Frontend (React/Vite)
│   ├── Dockerfile                # Container definition
│   ├── nginx.conf                # Nginx configuration
│   ├── package.json              # Dependencies
│   ├── vite.config.ts            # Vite configuration
│   ├── public/                   # Static assets
│   │   └── images/               # Static images
│   └── src/
│       ├── components/           # React components
│       ├── context/              # React contexts
│       ├── hooks/                # Custom hooks
│       ├── icons/                # SVG icons
│       ├── layout/               # Layout components
│       ├── pages/                # Page components
│       ├── routes/               # Route definitions
│       ├── services/             # API services
│       └── types/                # TypeScript types
│
└── whatsnewasia_ref_revision/    # Documentation
    ├── README.md                 # This file
    ├── admin_fix.md              # Admin login fixes
    ├── deploy_info.md            # Deployment info
    ├── gcp_setup.md              # GCP setup guide
    ├── image_fix.md              # Image loading fixes
    └── ...                       # Other documentation
```

## Frontend Structure

### Layout System

The frontend uses two main layouts:

1. **FrontLayout** - Public website layout
   - Header with navigation
   - Dynamic content area
   - Footer with newsletter
   - Country/city-specific theming

2. **AppLayout** (AdminLayout) - Admin panel layout
   - Sidebar navigation
   - Top header with user menu
   - Protected routes
   - Dashboard widgets

### Route Structure

#### Public Routes (`FrontApp.tsx`)

| Route Pattern | Description |
|---------------|-------------|
| `/` | Homepage |
| `/signin` | Login page |
| `/:country` | Country homepage |
| `/:country/:category` | Category listing |
| `/:country/:category/:slug` | Article detail |

The `PathResolver` component dynamically determines content based on URL segments.

#### Admin Routes (`AdminApp.tsx`)

| Route | Description | Access |
|-------|-------------|--------|
| `/admin` | Dashboard | All authenticated |
| `/admin/mst_article` | Article management | All authenticated |
| `/admin/mst_article/create` | Create article | All authenticated |
| `/admin/mst_article/edit/:id` | Edit article | All authenticated |
| `/admin/mst_locations` | Location management | All authenticated |
| `/admin/mst_categories` | Category management | Super admin |
| `/admin/mst_tags` | Tag management | Super admin |
| `/admin/mst_templates` | Template management | All authenticated |
| `/admin/media_library` | Media library | All authenticated |
| `/admin/users` | User management | Super admin |
| `/admin/subscribers` | Newsletter subscribers | All authenticated |
| `/admin/setting` | System settings | Super admin |

### Component Organization

```
src/components/
├── front/                  # Public site components
│   ├── Header.tsx          # Site header
│   ├── Footer.tsx          # Site footer
│   ├── Newsletter.tsx      # Newsletter signup
│   ├── ArticleCard.tsx     # Article preview card
│   ├── Image.tsx           # Lazy-loaded images
│   └── NavLogo.tsx         # Dynamic logo
│
├── header/                 # Header variations
├── WhatsnewasiaDash/       # Dashboard widgets
├── ProtectedRoute.tsx      # Auth guard
└── ui/                     # Reusable UI components
```

## Admin Panel

### Features

1. **Dashboard** (`/admin`)
   - Article statistics
   - Recent articles
   - Trending articles
   - Quick actions

2. **Article Management** (`/admin/mst_article`)
   - Create/edit articles
   - Rich text editor (Quill)
   - Featured image upload
   - Category/tag assignment
   - Location assignment
   - Draft/publish workflow
   - Scheduled publishing

3. **Media Library** (`/admin/media_library`)
   - Upload images
   - Browse uploaded assets
   - Image metadata editing
   - Used in article editor

4. **Template Management** (`/admin/mst_templates`)
   - Edit homepage layouts
   - Location-specific templates
   - Drag-and-drop article placement

5. **User Management** (`/admin/users`)
   - Create/edit users
   - Role assignment
   - Location restrictions

6. **Master Data**
   - Categories (`/admin/mst_categories`)
   - Tags (`/admin/mst_tags`)
   - Locations (`/admin/mst_locations`)

### User Roles

| Role | Description | Capabilities |
|------|-------------|--------------|
| `super_admin` | System administrator | Full access |
| `admin_country` | Country admin | Manage country content |
| `admin_city` | City admin | Manage city content |
| `admin_region` | Region admin | Manage region content |
| `editor` | Content editor | Create/edit articles |

## Authentication

### Flow

1. User submits credentials to `/api/auth/login`
2. Backend validates credentials with bcrypt
3. Backend generates JWT access token and refresh token
4. Tokens are set as httpOnly cookies
5. Frontend stores minimal user info in context
6. Protected routes check for valid cookies
7. Refresh token is used to get new access token

### Token Configuration

| Token | Expiry | Storage |
|-------|--------|---------|
| Access Token | 1 hour | httpOnly cookie |
| Refresh Token | 7 days | httpOnly cookie |

### Cookie Settings

```javascript
const cookieOptions = {
  httpOnly: true,
  secure: true,        // HTTPS only
  sameSite: "none",    // Required for cross-origin
};
```

### Admin Credentials (Default)

| Field | Value |
|-------|-------|
| Email | `super_admin@admin.com` |
| Password | `12345678` |

To create the admin user, run:
```bash
npx sequelize-cli db:seed --seed 20250922090055-demo-user.cjs
```

## Database Schema

### Core Tables

| Table | Description |
|-------|-------------|
| `users` | User accounts |
| `articles` | Article metadata |
| `article_versions` | Article content versions |
| `category` | Content categories |
| `tags` | Article tags |
| `article_tags` | Article-tag relationships |
| `country` | Countries |
| `city` | Cities |
| `region` | Regions |
| `asset_media` | Uploaded images |
| `article_templating` | Page templates |
| `subscribers` | Newsletter subscribers |

### Key Relationships

```
articles
├── belongs_to category
├── belongs_to country
├── belongs_to city
├── belongs_to region
├── belongs_to asset_media (featured_image)
├── has_many article_versions
└── has_many article_tags

article_versions
├── belongs_to articles
└── belongs_to asset_media (featured_image)

country
├── has_many city
└── belongs_to asset_media (site_logo)

city
├── belongs_to country
├── has_many region
└── belongs_to asset_media (site_logo)
```

### Migrations

Run migrations:
```bash
cd whatsnewasia_be_revision
npx sequelize-cli db:migrate
```

Run seeders:
```bash
npx sequelize-cli db:seed:all
```

Undo migrations:
```bash
npx sequelize-cli db:migrate:undo:all
```

## Local Development

### Prerequisites

- Node.js 20+
- MySQL 8 (or Cloud SQL Proxy)
- npm or yarn

### Backend Setup

```bash
cd whatsnewasia_be_revision

# Install dependencies
npm install

# Create .env file
cp .env.sample .env
# Edit .env with your database credentials

# Start Cloud SQL Proxy (if using Cloud SQL)
cloud-sql-proxy godaexplore:asia-southeast2:gda-wna-mysql8 --port=3306

# Run migrations
npm run db:migrate

# Run seeders
npm run db:seed

# Start development server
npm run dev
```

Backend runs on: http://localhost:7777

### Frontend Setup

```bash
cd whatsnewasia_fe_revision

# Install dependencies
npm install

# Create .env file
echo "VITE_WHATSNEW_BACKEND_URL=http://localhost:7777" > .env

# Start development server
npm run dev
```

Frontend runs on: http://localhost:5173

### Environment Files

**Backend `.env`**:
```env
NODE_ENV=development
DATABASE_USER=root
DATABASE_PASSWORD=your_password
DATABASE=whatsnewasia
DATABASE_HOST=127.0.0.1
JWT_SECRET_DEV=your_jwt_secret
JWT_REFRESH_SECRET_DEV=your_refresh_secret
```

**Frontend `.env`**:
```env
VITE_WHATSNEW_BACKEND_URL=http://localhost:7777
```

**Frontend `.env.production`**:
```env
VITE_WHATSNEW_BACKEND_URL=https://whatsnewasia-backend-850916858221.asia-southeast2.run.app
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

## Deployment

### Deploy Backend

```bash
cd /path/to/whatsnewasia

gcloud run deploy whatsnewasia-backend \
  --source ./whatsnewasia_be_revision \
  --region asia-southeast2 \
  --allow-unauthenticated \
  --set-env-vars "\
NODE_ENV=production,\
DB_HOST=/cloudsql/godaexplore:asia-southeast2:gda-wna-mysql8,\
DB_USER=root,\
DB_PASS=dsKqGW0nA6sEwJK59rZk,\
DB_NAME=whatsnewasia,\
JWT_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=,\
JWT_REFRESH_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=" \
  --add-cloudsql-instances godaexplore:asia-southeast2:gda-wna-mysql8
```

### Deploy Frontend

```bash
cd /path/to/whatsnewasia

gcloud run deploy whatsnewasia-frontend \
  --source ./whatsnewasia_fe_revision \
  --region asia-southeast2 \
  --allow-unauthenticated
```

### View Logs

```bash
# Backend logs
gcloud run services logs read whatsnewasia-backend --region asia-southeast2 --limit=50

# Frontend logs
gcloud run services logs read whatsnewasia-frontend --region asia-southeast2 --limit=50
```

### Rollback

```bash
# List revisions
gcloud run revisions list --service=whatsnewasia-backend --region=asia-southeast2

# Rollback to specific revision
gcloud run services update-traffic whatsnewasia-backend \
  --region=asia-southeast2 \
  --to-revisions=whatsnewasia-backend-00001-abc=100
```

## When to Redeploy

### Backend Redeploy Required

| Change Type | Examples | Command |
|-------------|----------|---------|
| API changes | New endpoints, modified responses | `gcloud run deploy whatsnewasia-backend --source ./whatsnewasia_be_revision --region asia-southeast2 --allow-unauthenticated --add-cloudsql-instances godaexplore:asia-southeast2:gda-wna-mysql8` |
| Service logic | Business logic changes | Same as above |
| Middleware | Auth, validation changes | Same as above |
| Model changes | New fields, relationships | Run migrations first, then redeploy |
| Dependencies | package.json changes | Same as above |
| Environment vars | New env vars needed | Add `--set-env-vars KEY=value` |

### Frontend Redeploy Required

| Change Type | Examples | Command |
|-------------|----------|---------|
| UI changes | Component updates, styling | `gcloud run deploy whatsnewasia-frontend --source ./whatsnewasia_fe_revision --region asia-southeast2 --allow-unauthenticated` |
| Route changes | New pages, route structure | Same as above |
| API integration | New API calls, service updates | Same as above |
| Dependencies | package.json changes | Same as above |
| Static assets | New images in public/ | Same as above |
| Build config | vite.config changes | Same as above |

### No Redeploy Required

| Change Type | Reason |
|-------------|--------|
| Database content | Data changes via API/admin |
| GCS images | Directly uploaded to bucket |
| Database schema | Migrations run separately |
| Seeder data | Run via Cloud SQL Proxy |

### Full Redeploy Commands

**Backend:**
```bash
gcloud run deploy whatsnewasia-backend \
  --source ./whatsnewasia_be_revision \
  --region asia-southeast2 \
  --allow-unauthenticated \
  --set-env-vars "NODE_ENV=production,DB_HOST=/cloudsql/godaexplore:asia-southeast2:gda-wna-mysql8,DB_USER=root,DB_PASS=dsKqGW0nA6sEwJK59rZk,DB_NAME=whatsnewasia,JWT_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=,JWT_REFRESH_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=" \
  --add-cloudsql-instances godaexplore:asia-southeast2:gda-wna-mysql8
```

**Frontend:**
```bash
gcloud run deploy whatsnewasia-frontend \
  --source ./whatsnewasia_fe_revision \
  --region asia-southeast2 \
  --allow-unauthenticated
```

## Environment Variables

### Backend

| Variable | Description | Example |
|----------|-------------|---------|
| `NODE_ENV` | Environment mode | `production` |
| `DB_HOST` | Database host | `/cloudsql/project:region:instance` |
| `DB_USER` | Database username | `root` |
| `DB_PASS` | Database password | `password` |
| `DB_NAME` | Database name | `whatsnewasia` |
| `JWT_SECRET` | JWT signing secret | `base64-encoded-secret` |
| `JWT_REFRESH_SECRET` | Refresh token secret | `base64-encoded-secret` |
| `FRONTEND_URL` | Frontend URL for CORS | `https://frontend.run.app` |

### Frontend

| Variable | Description | Example |
|----------|-------------|---------|
| `VITE_WHATSNEW_BACKEND_URL` | Backend API URL | `https://backend.run.app` |
| `VITE_IMAGE_URL` | Image CDN URL | `https://storage.googleapis.com/bucket/path` |


## Troubleshooting

### Common Issues

#### 1. "secretOrPrivateKey must have a value"
- **Cause**: JWT_SECRET not set in production
- **Fix**: Ensure `JWT_SECRET` (not `JWT_SECRET_DEV`) is set in Cloud Run env vars

#### 2. "No routes matched location '/signin'"
- **Cause**: /signin route missing from AdminApp
- **Fix**: Add SignIn route to AdminApp.tsx

#### 3. Cookies not being sent
- **Cause**: sameSite cookie setting
- **Fix**: Set `sameSite: "none"` for cross-origin cookies

#### 4. Images not loading
- **Cause**: Missing asset_media entries or wrong paths
- **Fix**: Run image seeders, ensure paths match GCS structure

#### 5. Cloud Run deployment fails
- **Cause**: Container startup timeout
- **Fix**: Check logs, ensure app listens on PORT env var

### Useful Commands

```bash
# Check service status
gcloud run services describe whatsnewasia-backend --region asia-southeast2

# View recent logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=whatsnewasia-backend" --limit=20

# Connect to Cloud SQL locally
cloud-sql-proxy godaexplore:asia-southeast2:gda-wna-mysql8 --port=3306

# Test API endpoint
curl https://whatsnewasia-backend-850916858221.asia-southeast2.run.app/api/article?limit=1
```

## Related Documentation

- [admin_fix.md](admin_fix.md) - Admin login fixes
- [image_fix.md](image_fix.md) - Image loading fixes
- [deploy_fix.md](deploy_fix.md) - Deployment fixes
- [gcp_setup.md](gcp_setup.md) - GCP setup guide
- [deploy_info.md](deploy_info.md) - Deployment commands


# WhatsnewAsia Local Development Setup

This guide covers setting up the WhatsnewAsia application for local development.

---

## Prerequisites

- **Node.js** 20+
- **MySQL** 8.0
- **Redis** (optional, for caching)

---

## Quick Start

### 1. Backend Setup (Port 7777)

```bash
cd whatsnewasia_be_revision

# Install dependencies
npm install

# Create the database (requires MySQL running)
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS whatsnewasia;"

# Run migrations
npm run db:migrate

# (Optional) Seed the database with sample data
npm run db:seed

# Start the development server
npm run dev
```

### 2. Frontend Setup (Port 5173)

```bash
cd whatsnewasia_fe_revision

# Install dependencies
npm install

# Start the development server
npm run dev
```

### 3. Access the Application

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:7777 |
| Admin Panel | http://localhost:5173/admin |

---

## Environment Configuration

### Backend (.env)

Location: `whatsnewasia_be_revision/.env`

```env
# Server
PORT=7777
URL=http://localhost
NODE_ENV=development

# Database Connection
DATABASE_HOST=localhost
DATABASE_PORT=3306
DATABASE_USER=root
DATABASE_PASSWORD=your_password_here

# Database Names
DATABASE=whatsnewasia
DATABASE_PRODUCTION=whatsnewasia
DATABASE_TEST=whatsnewasia_test

# Connection Pool
DB_POOL_MAX=10
DB_POOL_MIN=0
DB_POOL_IDLE=10000
DB_POOL_ACQUIRE=30000
DB_TIMEZONE="+07:00"

# Authentication
JWT_SECRET=your_jwt_secret_here
JWT_REFRESH_SECRET=your_jwt_refresh_secret_here

# CORS
WHITELIST_CORS_ORIGIN=[http://localhost:3000,http://localhost:7777]
FRONTEND_URL=http://localhost:5173,http://localhost:3000,http://localhost:5175

# Redis (optional)
REDIS_PORT=6379
REDIS_HOST=127.0.0.1

# Image Storage (local development uses backend server)
IMAGE_URL=http://localhost:7777
```

### Frontend (.env)

Location: `whatsnewasia_fe_revision/.env`

```env
VITE_WHATSNEW_BACKEND_URL=http://localhost:7777
VITE_SITE_URL=http://localhost:5173
VITE_IMAGE_URL=http://localhost:7777
VITE_HMR=true
VITE_HOST=127.0.0.1
```

---

## Database Commands

### Migrations

```bash
cd whatsnewasia_be_revision

# Run all pending migrations
npm run db:migrate

# Undo last migration
npm run db:migrate:undo

# Undo all migrations
npm run db:migrate:undo:all
```

### Seeds

```bash
cd whatsnewasia_be_revision

# Run all seeds
npm run db:seed

# Undo last seed
npm run db:seed:undo

# Undo all seeds
npm run db:seed:undo:all
```

---

## Available Scripts

### Backend

| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server with nodemon |
| `npm run start` | Start production server |
| `npm run db:migrate` | Run database migrations |
| `npm run db:seed` | Seed the database |

### Frontend

| Command | Description |
|---------|-------------|
| `npm run dev` | Start Vite dev server with HMR |
| `npm run build` | Build for production |
| `npm run build:ssr` | Build with SSR support |
| `npm run preview` | Preview production build |
| `npm run lint` | Run ESLint |

---

## Local vs Production Comparison

| Setting | Local | Production (GCP) |
|---------|-------|------------------|
| Frontend URL | http://localhost:5173 | https://whatsnewasia-frontend-xxx.run.app |
| Backend URL | http://localhost:7777 | https://whatsnewasia-backend-xxx.run.app |
| Database Host | localhost | /cloudsql/PROJECT:region:instance |
| Image Storage | http://localhost:7777 | https://storage.googleapis.com/gda_p01_storage/gda_wna_images |
| Redis | localhost:6379 | Cloud Memorystore (if configured) |

---

## Troubleshooting

### MySQL Connection Issues

```bash
# Check if MySQL is running
mysql -u root -p -e "SELECT 1;"

# Create database if missing
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS whatsnewasia CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### Port Already in Use

```bash
# Find process using port 7777
lsof -i :7777

# Kill the process
kill -9 <PID>
```

### Frontend Can't Connect to Backend

1. Ensure backend is running on port 7777
2. Check `VITE_WHATSNEW_BACKEND_URL` in frontend `.env`
3. Verify CORS settings in backend `.env`

### Database Migration Errors

```bash
# Check migration status
npx sequelize-cli db:migrate:status

# Reset and re-run migrations
npm run db:migrate:undo:all
npm run db:migrate
```

---

## Important Notes

- **Local development does NOT affect GCP** - The `.env` file points to `localhost`, not Cloud SQL
- **Images in local dev** - Uploaded images are stored in `whatsnewasia_be_revision/uploads/`
- **Redis is optional** - The app works without Redis but caching won't function
- **SSR mode** - For SSR development, use `npm run ssr` on the backend

---

## File Locations

| Purpose | File |
|---------|------|
| Backend environment | `whatsnewasia_be_revision/.env` |
| Frontend environment (dev) | `whatsnewasia_fe_revision/.env` |
| Frontend environment (prod) | `whatsnewasia_fe_revision/.env.production` |
| Database config | `whatsnewasia_be_revision/config/config.js` |
| Vite config | `whatsnewasia_fe_revision/vite.config.ts` |

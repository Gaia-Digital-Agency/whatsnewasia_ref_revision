# GCP Deployment Guide: WhatsNewAsia

This guide deploys the frontend and backend to **Google Cloud Run**.

## Prerequisites

1. [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed
2. A GCP project with billing enabled

## 1. Initial Setup

```bash
# Login to GCP
gcloud auth login

# Set your project (replace with your project ID)
gcloud config set project YOUR_PROJECT_ID

# Enable required APIs
gcloud services enable \
    run.googleapis.com \
    cloudbuild.googleapis.com \
    artifactregistry.googleapis.com \
    sqladmin.googleapis.com

# Create Docker repository
gcloud artifacts repositories create whatsnewasia-repo \
    --repository-format=docker \
    --location=us-central1 \
    --description="WhatsNewAsia Docker images"

# Configure Docker authentication
gcloud auth configure-docker us-central1-docker.pkg.dev
```

## 2. Create Cloud SQL Instance (MySQL)

```bash
# Create MySQL 8 instance (db-f1-micro is cheapest, use db-g1-small for production)
gcloud sql instances create whatsnewasia-db \
    --database-version=MYSQL_8_0 \
    --tier=db-f1-micro \
    --region=us-central1 \
    --root-password=YOUR_DB_ROOT_PASSWORD

# Create the database
gcloud sql databases create whatsnewasia --instance=whatsnewasia-db

# Create application user
gcloud sql users create appuser \
    --instance=whatsnewasia-db \
    --password=YOUR_APP_USER_PASSWORD
```

## 3. Deploy Backend

From the project root directory:

```bash
cd /path/to/whatsnewasia

# Build and push backend image
gcloud builds submit ./backend \
    --tag us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/backend

# Deploy to Cloud Run
gcloud run deploy whatsnewasia-backend \
    --image us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/backend \
    --platform managed \
    --region us-central1 \
    --allow-unauthenticated \
    --add-cloudsql-instances YOUR_PROJECT_ID:us-central1:whatsnewasia-db \
    --memory 512Mi \
    --set-env-vars "\
NODE_ENV=production,\
DATABASE_HOST=/cloudsql/YOUR_PROJECT_ID:us-central1:whatsnewasia-db,\
DATABASE_USER=appuser,\
DATABASE_PASSWORD=YOUR_APP_USER_PASSWORD,\
DATABASE_PRODUCTION=whatsnewasia,\
JWT_SECRET=your-secure-jwt-secret-min-32-chars,\
JWT_REFRESH_SECRET=your-secure-refresh-secret-min-32-chars,\
ENCRYPT_KEY=your-encrypt-key,\
FRONTEND_URL=https://YOUR_FRONTEND_URL"
```

**Note the backend URL** from the output (e.g., `https://whatsnewasia-backend-xxxxx-uc.a.run.app`)

## 4. Run Database Migrations

Connect to Cloud SQL and run migrations:

```bash
# Option 1: Use Cloud SQL Proxy locally
# Download from: https://cloud.google.com/sql/docs/mysql/sql-proxy

# Start proxy in a separate terminal
cloud-sql-proxy YOUR_PROJECT_ID:us-central1:whatsnewasia-db

# In another terminal, run migrations
cd backend/whatsnewasia_be_revision
DATABASE_HOST=127.0.0.1 \
DATABASE_USER=appuser \
DATABASE_PASSWORD=YOUR_APP_USER_PASSWORD \
DATABASE=whatsnewasia \
npm run db:migrate

# Run seeders if needed
npm run db:seed
```

## 5. Deploy Frontend

First, update the frontend environment with your backend URL:

```bash
# Create production env file
cat > frontend/whatsnewasia_fe_revision/.env.production << EOF
VITE_WHATSNEW_BACKEND_URL=https://whatsnewasia-backend-xxxxx-uc.a.run.app
EOF
```

Then deploy:

```bash
cd /path/to/whatsnewasia

# Build and push frontend image
gcloud builds submit ./frontend \
    --tag us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/frontend

# Deploy to Cloud Run
gcloud run deploy whatsnewasia-frontend \
    --image us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/frontend \
    --platform managed \
    --region us-central1 \
    --allow-unauthenticated \
    --memory 256Mi
```

## 6. Update Backend CORS

After deploying frontend, update backend with the frontend URL:

```bash
gcloud run services update whatsnewasia-backend \
    --region us-central1 \
    --update-env-vars "FRONTEND_URL=https://whatsnewasia-frontend-xxxxx-uc.a.run.app,WHITELIST_CORS_ORIGIN=https://whatsnewasia-frontend-xxxxx-uc.a.run.app"
```

## Quick Reference Commands

```bash
# View logs
gcloud run services logs read whatsnewasia-backend --region us-central1
gcloud run services logs read whatsnewasia-frontend --region us-central1

# Get service URLs
gcloud run services describe whatsnewasia-backend --region us-central1 --format='value(status.url)'
gcloud run services describe whatsnewasia-frontend --region us-central1 --format='value(status.url)'

# Redeploy after code changes
gcloud builds submit ./backend --tag us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/backend
gcloud run deploy whatsnewasia-backend --image us-central1-docker.pkg.dev/YOUR_PROJECT_ID/whatsnewasia-repo/backend --region us-central1

# Delete services (cleanup)
gcloud run services delete whatsnewasia-backend --region us-central1
gcloud run services delete whatsnewasia-frontend --region us-central1
gcloud sql instances delete whatsnewasia-db
```

## Cost Optimization Tips

1. Cloud Run only charges when handling requests
2. Use `--min-instances=0` (default) for auto-scaling to zero
3. Use `db-f1-micro` for Cloud SQL during testing
4. Consider using Cloud SQL Auth Proxy in production for better security

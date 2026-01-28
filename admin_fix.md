# Admin Login Fix Documentation

This document describes the issues encountered and fixes applied to enable admin login to the CMS.

## Issues and Fixes

### 1. JWT Secret Configuration Bug

**Problem:** Login attempts failed with error:
```
Error: secretOrPrivateKey must have a value
```

**Cause:** The `jwtoken.js` helper was using `JWT_SECRET_DEV` environment variable for production instead of `JWT_SECRET`.

**File:** `whatsnewasia_be_revision/src/helpers/jwtoken.js`

**Fix:** Corrected the environment variable names for production:

```javascript
let jwt_secret_key, jwt_refresh_secret_key;
if (process.env.NODE_ENV === "development") {
  jwt_secret_key = process.env.JWT_SECRET_DEV;
  jwt_refresh_secret_key = process.env.JWT_REFRESH_SECRET_DEV;
} else {
  jwt_secret_key = process.env.JWT_SECRET;
  jwt_refresh_secret_key = process.env.JWT_REFRESH_SECRET;
}
```

### 2. React Router Missing Route

**Problem:** After navigating to `/signin` from the admin page, console showed:
```
No routes matched location "/signin"
```

**Cause:** The `/signin` route was defined in `FrontApp.tsx` but not in `AdminApp.tsx`. When accessing the site from `/admin`, the app loads `AdminApp` which didn't have the `/signin` route defined.

**File:** `whatsnewasia_fe_revision/src/routes/AdminApp.tsx`

**Fix:** Added the `/signin` route to `AdminApp.tsx`:

```tsx
const SignIn = lazy(() => import("../pages/AuthPages/SignIn"))

export const adminRoutes: RouteObject[] = [
    { path: "/signin", element: <Suspense fallback={<></>}><SignIn /></Suspense> },
    {
        path: "/admin",
        element: <AdminLayout />,
        children: [
            // ... existing admin routes
        ]
    }
]
```

### 3. Cross-Origin Cookie Issue

**Problem:** After successful login (JWT token generated), the user was not authenticated when redirected to the CMS. Cookies were not being sent with subsequent requests.

**Cause:** The `sameSite` cookie option was set to `"strict"`, which prevents cookies from being sent on cross-origin requests. Since the frontend and backend are on different Cloud Run URLs, cookies weren't being included.

**File:** `whatsnewasia_be_revision/src/controllers/auth.controller.js`

**Fix:** Changed `sameSite` from `"strict"` to `"none"`:

```javascript
const cookieOptions = {
  httpOnly: true,
  secure: true,
  sameSite: "none", // Required for cross-origin cookies
};
```

## Deployment Steps After Fixes

After making the above changes, the services were redeployed:

### Backend Redeployment
```bash
cd whatsnewasia_be_revision
gcloud run deploy whatsnewasia-backend \
  --source . \
  --region asia-southeast2 \
  --allow-unauthenticated \
  --set-env-vars NODE_ENV=production,DB_HOST=/cloudsql/godaexplore:asia-southeast2:gda-wna-mysql8,DB_USER=root,DB_PASS=dsKqGW0nA6sEwJK59rZk,DB_NAME=gda_wna_db,JWT_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=,JWT_REFRESH_SECRET=iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo= \
  --add-cloudsql-instances godaexplore:asia-southeast2:gda-wna-mysql8
```

### Frontend Redeployment
```bash
cd whatsnewasia_fe_revision
gcloud run deploy whatsnewasia-frontend \
  --source . \
  --region asia-southeast2 \
  --allow-unauthenticated
```

## Admin Credentials

- **Email:** super_admin@admin.com
- **Password:** 12345678

## User Seeder

If the admin user doesn't exist, run the seeder:

```bash
# Start Cloud SQL Proxy first
cloud-sql-proxy godaexplore:asia-southeast2:gda-wna-mysql8 --port=3306

# In another terminal, run the seeder
cd whatsnewasia_be_revision
npx sequelize-cli db:seed --seed 20250922090055-demo-user.cjs
```

Note: If you see a "Duplicate entry" error, it means the user already exists in the database.

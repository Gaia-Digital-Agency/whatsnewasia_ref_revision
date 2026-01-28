# Minor Fixes

## 1. SSR Path Guards - Backend Startup Fix

### Issue
Backend deployment to Cloud Run was failing with:
```
TypeError [ERR_INVALID_ARG_TYPE]: The "paths[2]" argument must be of type string. Received undefined
```

### Cause
The `app.js` was trying to resolve `process.env.FRONTEND_PATH` which is not set in production (since frontend and backend are separate Cloud Run services).

### Solution
Added guards throughout `app.js` to make SSR/Vite functionality optional when `FRONTEND_PATH` is not set.

**File:** `whatsnewasia_be_revision/app.js`

```javascript
// SSR paths - only used if FRONTEND_PATH is provided
const frontendPath = process.env.FRONTEND_PATH
  ? path.resolve(__dirname, '..', process.env.FRONTEND_PATH)
  : null;
const templatePath = frontendPath
  ? (isProd
    ? path.resolve(frontendPath, 'index.html')
    : path.resolve(frontendPath, 'index.html'))
  : null;

// Vite setup guard
let vite;
if (!isProd && frontendPath) {
  const { createServer: createViteServer } = await import('vite');
  vite = await createViteServer({
    server: { middlewareMode: true },
    appType: 'custom',
    root: frontendPath,
  });
}

// Static file serving guard
if (!isProd && vite) {
  app.use(vite.middlewares);
} else if (isProd && frontendPath) {
  app.use((await import('compression')).default());
  app.use(
    (await import('serve-static')).default(path.resolve(frontendPath, 'dist'), {
      index: false,
    })
  );
}

// SSR route handler guard
app.use('*', async (req, res, next) => {
  if (req.originalUrl.startsWith('/api')) {
    return next();
  }

  // Skip SSR when frontendPath is not configured
  if (process.env.DISABLE_SSR === 'true' || !frontendPath) {
    return res.status(200).json({ message: 'API server running. Use /api endpoints.' });
  }
  // ... rest of SSR handling
});
```

---

## 2. Database Connection Environment Variables

### Issue
Database connection was failing with "Database connection error" after deployment.

### Cause
Environment variables were set with wrong names:
- Set: `DB_HOST`, `DB_USER`, `DB_PASS`, `DB_NAME`
- Expected: `DATABASE_HOST`, `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_PRODUCTION`

### Solution
Updated Cloud Run environment variables to use correct names:

```bash
gcloud run services update whatsnewasia-backend \
  --region asia-southeast2 \
  --update-env-vars "DATABASE_HOST=/cloudsql/gda-p01:asia-southeast2:whatsnewasia-db-asia,DATABASE_USER=root,DATABASE_PASSWORD=dsKqGW0nA6sEwJK59rZk,DATABASE_PRODUCTION=whatsnewasia"
```

---

## 3. Cloud SQL Instance Path

### Issue
Connection to Cloud SQL was failing with `EIO` error.

### Cause
The Cloud SQL instance path was pointing to wrong project/instance:
- Wrong: `/cloudsql/godaexplore:asia-southeast2:gda-wna-mysql8`
- Correct: `/cloudsql/gda-p01:asia-southeast2:whatsnewasia-db-asia`

### Solution
```bash
gcloud run services update whatsnewasia-backend \
  --region asia-southeast2 \
  --update-env-vars "DATABASE_HOST=/cloudsql/gda-p01:asia-southeast2:whatsnewasia-db-asia"
```

---

## 4. CORS Configuration

### Issue
Frontend was getting CORS errors:
```
Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource
CORS header 'Access-Control-Allow-Origin' missing
```

### Cause
`FRONTEND_URL` environment variable was not set on the backend, so the CORS middleware didn't know which origins to allow.

### Solution
Added the frontend URL to the backend environment variables:

```bash
gcloud run services update whatsnewasia-backend \
  --region asia-southeast2 \
  --update-env-vars "FRONTEND_URL=https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app"
```

### Verification
Test CORS preflight:
```bash
curl -s -I -X OPTIONS \
  -H "Origin: https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app" \
  -H "Access-Control-Request-Method: GET" \
  "https://whatsnewasia-backend-850916858221.asia-southeast2.run.app/api/location"
```

Expected response should include:
```
access-control-allow-origin: https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app
access-control-allow-credentials: true
```

---

## Current Backend Environment Variables

After all fixes, the backend should have these environment variables:

| Variable | Value |
|----------|-------|
| `NODE_ENV` | `production` |
| `DATABASE_HOST` | `/cloudsql/gda-p01:asia-southeast2:whatsnewasia-db-asia` |
| `DATABASE_USER` | `root` |
| `DATABASE_PASSWORD` | `dsKqGW0nA6sEwJK59rZk` |
| `DATABASE_PRODUCTION` | `whatsnewasia` |
| `JWT_SECRET` | `iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=` |
| `JWT_REFRESH_SECRET` | `iKMhOSum5wnyxzVre8N+mBJUuSRPqPBuc8J8I0/2tWo=` |
| `FRONTEND_URL` | `https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app` |

## Verify Configuration
```bash
gcloud run services describe whatsnewasia-backend \
  --region asia-southeast2 \
  --format='yaml(spec.template.spec.containers[0].env)'
```

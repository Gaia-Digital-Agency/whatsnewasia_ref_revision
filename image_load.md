# Image Loading Fix

## Problem
Images were not loading on the site after migration. The image files were moved from local storage to Google Cloud Storage (GCS) bucket.

## Root Cause
The application was configured to load images from the backend API URL (`VITE_WHATSNEW_BACKEND_URL`), but images were now stored in a separate GCS bucket at:
```
https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

## Solution
Added a new environment variable `VITE_IMAGE_URL` to point to the GCS bucket and updated all image-related code to use this variable.

### 1. Environment Variables Updated

**`.env.production`**
```
VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images
```

**`.env` and `.env.sample`** (for local development)
```
VITE_IMAGE_URL=http://localhost:7777
```

### 2. Files Modified

| File | Change |
|------|--------|
| `src/hooks/useArticle.ts` | Added `IMAGE_URL` constant, updated `getFeaturedImageUrl()` to use it |
| `src/components/front/Image.tsx` | Added `IMAGE_URL` constant for placeholder fallback |
| `src/pages/Front/Templates/Single.tsx` | Added `IMAGE_URL` for OG/Twitter meta image tags |
| `src/pages/Front/Templates/SingleEvent.tsx` | Added `IMAGE_URL` for OG/Twitter meta image tags |
| `src/pages/Front/Templates/SingleJob.tsx` | Added `IMAGE_URL` for OG/Twitter meta image tags |
| `src/pages/Front/Templates/SingleHousing.tsx` | Added `IMAGE_URL` for OG/Twitter meta image tags, updated `generateImageUrl()` |
| `src/pages/Front/Templates/Housing.tsx` | Added `IMAGE_URL`, updated `generateImageUrl()` |
| `src/pages/Front/Templates/Events.tsx` | Added `IMAGE_URL`, updated `generateImageUrl()` |

### 3. Code Pattern Used

Each file follows this pattern:
```typescript
const API_URL = import.meta.env.VITE_WHATSNEW_BACKEND_URL
const IMAGE_URL = import.meta.env.VITE_IMAGE_URL || API_URL
```

The fallback to `API_URL` ensures backward compatibility if `VITE_IMAGE_URL` is not set.

## GCS Bucket Details
- **Bucket Name:** gda_p01_storage
- **Folder:** gda_wna_images
- **Full URL:** https://storage.googleapis.com/gda_p01_storage/gda_wna_images
- **Console:** https://console.cloud.google.com/storage/browser/gda_p01_storage/gda_wna_images?project=gda-p01

## Deployment
After making these changes, redeploy the frontend:

```bash
# Build and push frontend image
gcloud builds submit ./frontend/whatsnewasia_fe_revision \
  --tag asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region=asia-southeast2

# Deploy to Cloud Run
gcloud run deploy whatsnewasia-frontend \
  --image asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region asia-southeast2
```

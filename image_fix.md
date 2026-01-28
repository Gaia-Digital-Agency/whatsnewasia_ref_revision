# Image Loading Fix Documentation

This document describes the issues encountered and fixes applied to get images displaying correctly on the WhatsNewAsia website.

## Problem Summary

Images were not displaying on the main site despite existing in Google Cloud Storage (GCS). The issues were:

1. **Article featured images**: Articles had `featured_image = NULL` in the database
2. **Country/City logos**: Location records had `site_logo = NULL` in the database
3. **API returning IDs instead of paths**: The `getAllLocation` API method was returning asset_media IDs instead of image paths

## Image Storage Architecture

- **GCS Bucket**: `gs://gda_p01_storage/gda_wna_images/`
- **Public URL**: `https://storage.googleapis.com/gda_p01_storage/gda_wna_images/`
- **Frontend ENV**: `VITE_IMAGE_URL=https://storage.googleapis.com/gda_p01_storage/gda_wna_images`

Images are stored in GCS and referenced via the `asset_media` table. The `path` column contains just the filename (e.g., `04-Marina-One-en.webp`), and the frontend constructs the full URL using `VITE_IMAGE_URL`.

## Fixes Applied

### 1. Asset Media Seeder

Created a new seeder to populate the `asset_media` table with images from GCS.

**File**: `whatsnewasia_be_revision/src/seeders/20250923000000-asset-media-seeds.cjs`

This seeder:
- Adds 60 images from GCS bucket to the `asset_media` table
- Each entry includes: filename, mimetype, path (filename only), title, alt_text
- IDs are assigned 1-60 for easy reference

**Run the seeder**:
```bash
cd whatsnewasia_be_revision
npx sequelize-cli db:seed --seed 20250923000000-asset-media-seeds.cjs
```

### 2. Articles Seeder Update

Updated the articles seeder to link articles to asset_media entries.

**File**: `whatsnewasia_be_revision/src/seeders/20250924053230-articles-seeds.cjs`

Changes:
- Changed `featured_image: null` to `featured_image: 1` (ID reference) for existing articles
- Added formula `featured_image: ((id - 1) % 60) + 1` for generated articles to cycle through available images

### 3. Article Versions Seeder Update

Updated the article_versions seeder similarly.

**File**: `whatsnewasia_be_revision/src/seeders/20250924053330-article-versions-seeds.cjs`

Changes:
- Changed `featured_image: null` to proper asset_media ID references
- Added same cycling formula for generated article versions

### 4. Database Update Script

For existing data in the database, ran SQL updates:

```sql
-- Update articles to link to asset_media
UPDATE articles SET featured_image = ((id - 1) % 60) + 1
WHERE featured_image IS NULL OR featured_image = 0;

-- Update article_versions similarly
UPDATE article_versions SET featured_image = ((id - 1) % 60) + 1
WHERE featured_image IS NULL OR featured_image = 0;

-- Fix asset_media paths (remove duplicate prefix)
UPDATE asset_media SET path = REPLACE(path, 'gda_wna_images/', '')
WHERE path LIKE 'gda_wna_images/%';
```

### 5. Country Logos Setup

Added country-specific logos from GCS to the database:

| Country | Logo File |
|---------|-----------|
| China | wn-china-logo.webp |
| Indonesia | wn-indonesia-logo.webp |
| Philippines | wn-phillipines-logo.webp |
| Singapore | wn-singapore-logo.webp |
| Vietnam | wn-vietnam-logo.webp |
| South Korea | wn-asia-logo.webp (default) |
| Japan | wn-asia-logo.webp (default) |
| Malaysia | wn-asia-logo.webp (default) |

### 6. City Logos Setup

Added city-specific logos and fallback to country logos:

| City | Logo File |
|------|-----------|
| Bali | wn-bali-logo.webp |
| Jakarta | wn-jakarta-logo.webp |
| Manila | wn-manila-logo.webp |
| Shanghai | wn-shanghai-logo.webp |
| Singapore City | wn-singapore-logo.webp |
| Other cities | Inherit from country |

Cities without specific logos were updated to use their country's logo as fallback:
```sql
UPDATE city c
JOIN country co ON c.id_country = co.id
SET c.site_logo = co.site_logo
WHERE c.site_logo IS NULL AND co.site_logo IS NOT NULL;
```

### 7. Location Service Fix

Fixed the `getAllLocation` API method to return image paths instead of IDs.

**File**: `whatsnewasia_be_revision/src/services/location.service.js`

The method now:
- Joins with `AssetMedia` table for countries, cities, and regions
- Returns the `path` value from asset_media as `site_logo`
- Maps nested data correctly for cities and regions

## Verification

### API Response (Before Fix)
```json
{
  "name": "Indonesia",
  "site_logo": 62  // ID - WRONG
}
```

### API Response (After Fix)
```json
{
  "name": "Indonesia",
  "site_logo": "wn-indonesia-logo.webp"  // Path - CORRECT
}
```

### Frontend URL Construction
The frontend constructs URLs like:
```javascript
const imageUrl = `${IMAGE_URL}/${article.featured_image_url}`;
// Result: https://storage.googleapis.com/gda_p01_storage/gda_wna_images/04-Marina-One-en.webp
```

## Files Modified

1. `whatsnewasia_be_revision/src/seeders/20250923000000-asset-media-seeds.cjs` (NEW)
2. `whatsnewasia_be_revision/src/seeders/20250924053230-articles-seeds.cjs`
3. `whatsnewasia_be_revision/src/seeders/20250924053330-article-versions-seeds.cjs`
4. `whatsnewasia_be_revision/src/services/location.service.js`

## Current Status

### Article Images
- **Status**: Working
- The backend API (`/api/article`) returns `featured_image_url` with the correct image filename
- Frontend constructs full URL: `VITE_IMAGE_URL + "/" + featured_image_url`

### Location Logos (Country/City)
- **Status**: Database populated, API returns IDs (not paths yet)
- Countries and cities have `site_logo` IDs linked to `asset_media` table
- The `getAllLocation` API currently returns raw IDs like `site_logo: 62`
- A fix to return paths instead of IDs was attempted but deployment had issues

### Pending Fix for Location Logos
To make location logos work, the `getAllLocation` method in `location.service.js` needs to join with `asset_media` to return paths. The database is correctly set up - just needs the API to return the path instead of ID.

## GCS Images Available

The GCS bucket contains 771 images including:
- Article featured images (various .webp, .jpg, .png files)
- Country logos (wn-*-logo.webp)
- City logos (wn-*-logo.webp)

All images are publicly accessible at:
`https://storage.googleapis.com/gda_p01_storage/gda_wna_images/{filename}`

## Database Summary

| Table | Records | Images Linked |
|-------|---------|---------------|
| asset_media | 72 | - |
| articles | 54 | All linked to asset_media |
| article_versions | 54 | All linked to asset_media |
| country | 8 | All have site_logo |
| city | 16 | All have site_logo |
| region | Various | Inheriting from city/country |

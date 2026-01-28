# Logo Display Fix

## Issue
Country/city logos were not displaying on the frontend. The alt-text was visible but images were not loading.

## Root Cause
Two issues were identified:

1. **Backend API returning IDs instead of paths**: The `/api/location` endpoint was returning `site_logo` as an integer ID (e.g., `66`) instead of the actual image filename/path.

2. **Frontend using wrong base URL**: The `NavLogo.tsx` component was constructing image URLs using `API_URL` (backend URL) instead of `IMAGE_URL` (GCS URL).

## Solution

### 1. Backend Fix - location.service.js

Updated the `getAllLocation()` function to include AssetMedia joins and transform the response to return image paths instead of IDs.

**File:** `whatsnewasia_be_revision/src/services/location.service.js`

```javascript
async getAllLocation() {
  try {
    const vaData = await Country.findAll({
      attributes: ["id", "name", "slug", "timezone", "site_logo"],
      include: [
        {
          model: AssetMedia,
          as: "asset",
          attributes: ["path"],
        },
        {
          model: City,
          as: "cities",
          attributes: ["id", "name", "slug", "site_logo"],
          include: [
            {
              model: AssetMedia,
              as: "asset",
              attributes: ["path"],
            },
            {
              model: Region,
              as: "regions",
              attributes: ["id", "name", "slug", "site_logo"],
              include: [
                {
                  model: AssetMedia,
                  as: "asset",
                  attributes: ["path"],
                },
              ],
            },
          ],
        },
      ],
    });

    // Transform to replace site_logo IDs with paths
    const result = vaData.map((country) => {
      const countryJson = country.toJSON();
      if (countryJson.asset?.path) {
        countryJson.site_logo = countryJson.asset.path;
      }
      delete countryJson.asset;

      if (countryJson.cities) {
        countryJson.cities = countryJson.cities.map((city) => {
          if (city.asset?.path) {
            city.site_logo = city.asset.path;
          }
          delete city.asset;

          if (city.regions) {
            city.regions = city.regions.map((region) => {
              if (region.asset?.path) {
                region.site_logo = region.asset.path;
              }
              delete region.asset;
              return region;
            });
          }
          return city;
        });
      }
      return countryJson;
    });

    return result;
  } catch (error) {
    logger.error("Failed to get all locations", { error: error.message });
    throw error;
  }
}
```

### 2. Frontend Fix - NavLogo.tsx

Updated the component to use `VITE_IMAGE_URL` (GCS URL) instead of `API_URL` for image sources.

**File:** `whatsnewasia_fe_revision/src/components/front/NavLogo.tsx`

```typescript
const API_URL = import.meta.env.VITE_WHATSNEW_BACKEND_URL
const IMAGE_URL = import.meta.env.VITE_IMAGE_URL || API_URL

// In renderImage function:
if(!image && defaultImage) {
  return <img src={defaultImage ? `${IMAGE_URL}/${defaultImage.url}` : '#'} ... />
}
return <img src={image ? `${IMAGE_URL}/${image.url}` : '#'} ... />
```

## API Response Comparison

### Before Fix
```json
{
  "id": 1,
  "name": "South Korea",
  "slug": "south-korea",
  "site_logo": 66,
  "cities": [...]
}
```

### After Fix
```json
{
  "id": 1,
  "name": "South Korea",
  "slug": "south-korea",
  "site_logo": "wn-asia-logo.webp",
  "cities": [...]
}
```

## Image URL Construction

The frontend now constructs logo URLs as:
```
https://storage.googleapis.com/gda_p01_storage/gda_wna_images/wn-asia-logo.webp
```

## Deployment

After making the changes:

1. **Deploy Backend:**
```bash
gcloud run deploy whatsnewasia-backend \
  --source ./whatsnewasia_be_revision \
  --region asia-southeast2 \
  --allow-unauthenticated \
  --add-cloudsql-instances gda-p01:asia-southeast2:whatsnewasia-db-asia
```

2. **Deploy Frontend:**
```bash
gcloud run deploy whatsnewasia-frontend \
  --source ./whatsnewasia_fe_revision \
  --region asia-southeast2 \
  --allow-unauthenticated
```

## Verification

Test the API to confirm paths are returned:
```bash
curl -s "https://whatsnewasia-backend-850916858221.asia-southeast2.run.app/api/location" | head -c 500
```

Expected output should show `"site_logo":"wn-asia-logo.webp"` instead of `"site_logo":66`.

---

## Homepage Logo Fix (Additional)

### Issue
After the above fix, country/city logos displayed correctly, but the homepage logo was still missing.

### Root Cause
The frontend relies on `window.__INITIAL_DATA__` injected by server-side rendering (SSR). However, SSR is disabled on the backend (because `FRONTEND_PATH` is not set in Cloud Run). Without SSR, the `initialData?.logo` is `undefined` on page load.

### Solution

#### 1. Create `/logo-header` template in database

Created seeder: `whatsnewasia_be_revision/src/seeders/20250128000000-logo-header-template.cjs`

```javascript
await queryInterface.bulkInsert("article_templating", [
  {
    url: "/logo-header",
    content: JSON.stringify({ url: "wn-asia-logo.webp", id: 66 }),
    template: "logo",
    isActive: true,
    createdBy: 1,
    updatedBy: 1,
    createdAt: new Date(),
    updatedAt: new Date(),
  },
]);
```

Run the seeder:
```bash
# Start Cloud SQL Proxy
cloud-sql-proxy gda-p01:asia-southeast2:whatsnewasia-db-asia --port=3306

# Run seeder
DATABASE_HOST=127.0.0.1 DATABASE_USER=root DATABASE_PASSWORD=dsKqGW0nA6sEwJK59rZk DATABASE=whatsnewasia \
npx sequelize-cli db:seed --seed 20250128000000-logo-header-template.cjs
```

#### 2. Update NavLogo to fetch logo via API

Modified `NavLogo.tsx` to fetch the logo template from the API when SSR data is not available:

```typescript
import { getTemplateByUrl } from "../../services/template.service"

const NavLogo: React.FC<NavLogoProps> = ({to = '/'}) => {
    const {initialData} = useHeaderContent()
    const [defaultImage, setDefaultImage] = useState<{url: string, id: number} | false>(initialData?.logo ?? false)

    // Fetch logo template if not provided by SSR
    useEffect(() => {
        if (!defaultImage) {
            (async () => {
                try {
                    const getImage = await getTemplateByUrl('/logo-header')
                    if(getImage?.status_code === 200 && getImage.data?.content) {
                        setDefaultImage(JSON.parse(getImage.data.content))
                    }
                } catch(e) {
                    console.log('Error fetching logo template:', e)
                }
            })()
        }
    }, [])

    // ... rest of component
}
```

### Key Changes
1. Changed `defaultImage` from a const to a state variable
2. Added useEffect to fetch `/logo-header` template via API when not provided by SSR
3. The API call fetches from `/api/templating/query?url=/logo-header`

### Deployment
After these changes, deploy both backend and frontend:

```bash
# Backend (for seeder data)
gcloud run deploy whatsnewasia-backend \
  --source ./whatsnewasia_be_revision \
  --region asia-southeast2 \
  --allow-unauthenticated \
  --add-cloudsql-instances gda-p01:asia-southeast2:whatsnewasia-db-asia

# Frontend (for NavLogo changes)
gcloud run deploy whatsnewasia-frontend \
  --source ./whatsnewasia_fe_revision \
  --region asia-southeast2 \
  --allow-unauthenticated
```

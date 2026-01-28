# Article Listing Date Fix

## Problem

Article listing dates were not displaying on the frontend. Pages affected:
- Events listing
- Deals listing
- Job Listing
- Directory listing
- Overseas listing

## Root Cause

The backend was formatting dates using Luxon with a non-standard format:

```javascript
localTimeCreated.toFormat("dd-MM-yyyy HH:mm:ss ZZZZ")
```

This produced strings like:
```
"27-01-2026 14:30:00 UTC+7"
```

JavaScript's `new Date()` constructor cannot parse this format, resulting in `Invalid Date`.

The frontend's `formatPublished()` function in `src/lib/utils/format.ts` does:
```typescript
const date = new Date(inputDate)  // Returns Invalid Date
```

When the date is invalid, `toLocaleDateString()` returns `"Invalid Date"` or the function returns `undefined`, causing no date to display.

## Solution

Changed the date format to ISO 8601, which JavaScript can natively parse:

```javascript
// Before
localTimeCreated.toFormat("dd-MM-yyyy HH:mm:ss ZZZZ")
// Output: "27-01-2026 14:30:00 UTC+7" (INVALID)

// After
localTimeCreated.toFormat("yyyy-MM-dd'T'HH:mm:ssZZ")
// Output: "2026-01-27T14:30:00+0700" (VALID ISO 8601)
```

## File Modified

**File:** `whatsnewasia_be_revision/src/services/article.service.js`

**Locations changed (5 occurrences):**
- Lines ~1176-1184
- Lines ~1570-1578
- Lines ~2136-2144
- Lines ~2500-2508
- Lines ~2818-2826

Each location had three date fields being formatted:
- `createdAt`
- `publishedAt`
- `updatedAt`

## Why ISO 8601?

ISO 8601 is the international standard for date/time representation:

| Feature | Old Format | ISO 8601 |
|---------|-----------|----------|
| JavaScript parseable | No | Yes |
| Cross-browser support | No | Yes |
| Timezone handling | Ambiguous | Clear |
| Example | `27-01-2026 14:30:00 UTC+7` | `2026-01-27T14:30:00+0700` |

## Verification

After deployment, verify dates display correctly:

1. Navigate to Events page - dates should show like "27 January 2026"
2. Navigate to Deals page - dates should display
3. Navigate to Job Listing - relative times like "2 days ago" should work
4. Check Directory and Overseas pages

## Related Frontend Code

The frontend parsing happens in:

**`whatsnewasia_fe_revision/src/lib/utils/format.ts`**

```typescript
export const formatPublished = (inputDate: string | undefined | number) => {
    if(!inputDate) return
    const date = new Date(inputDate)  // Now works with ISO 8601
    const formatted = date.toLocaleDateString('en-GB', {
        day: '2-digit',
        month: 'long',
        year: 'numeric',
    })
    return formatted  // Returns "27 January 2026"
}
```

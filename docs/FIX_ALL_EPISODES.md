# Fix: Get All Episodes Instead of Only 6

## Problem

The `/api/chapters/:bookId` endpoint was only returning 6 episodes instead of all available episodes.

## Root Cause

The `getChapters()` method in `src/services/Dramabox.js` was only fetching the first batch with a fixed `index: 1`, which returns approximately 5-6 episodes per batch.

## Solution

Updated the `getChapters()` method to:

1. Fetch the first batch to get the total episode count
2. Loop through all batches (incrementing index by 5) until all episodes are fetched
3. Remove duplicates and sort by chapterIndex
4. Add small delays between requests to avoid rate limiting

## Changes Made

### File: `src/services/Dramabox.js`

**Before:**

```javascript
async getChapters(bookId) {
  const data = await this.request("/drama-box/chapterv2/batch/load", {
    index: 1,  // Fixed index - only gets first batch!
    bookId,
  });

  const chapters = data?.data?.chapterList || [];
  return chapters; // Only returns ~6 episodes
}
```

**After:**

```javascript
async getChapters(bookId) {
  let allChapters = [];
  let totalChapters = 0;
  let currentIndex = 1;

  // Fetch first batch
  const firstBatch = await this.request(..., { index: 1, bookId });
  totalChapters = firstBatch?.data?.chapterCount || 0;
  allChapters.push(...firstBatch?.data?.chapterList || []);

  // Fetch remaining batches
  if (totalChapters > firstChapters.length) {
    currentIndex = 6;
    while (currentIndex <= totalChapters) {
      const batch = await this.request(..., { index: currentIndex, bookId });
      allChapters.push(...batch?.data?.chapterList || []);
      currentIndex += 5;
      await delay(500); // Rate limiting protection
    }
  }

  // Remove duplicates and sort
  const uniqueChapters = [...deduplicate and sort];
  return uniqueChapters; // Returns ALL episodes!
}
```

## Features Added

1. **Pagination Support**: Automatically fetches all batches of episodes
2. **Duplicate Removal**: Uses Map to remove Any duplicate chapters
3. **Sorting**: Sorts chapters by `chapterIndex` to ensure correct order
4. **Rate Limiting**: 500ms delay between batch requests
5. **Error Handling**: Continues fetching even if one batch fails
6. **Logging**: Console logs show progress and total chapters fetched
7. **Safety Limit**: MAX_BATCHES = 200 to prevent infinite loops

## Testing

After restarting the server:

```bash
npm run dev
```

Test the endpoint:

```bash
curl "http://localhost:3000/api/chapters/85522100014?lang=in"
```

Expected result:

- Before: ~6 episodes
- After: ALL episodes (likely 80-100+ episodes depending on the drama)

## Console Output Example

```
[Chapters] Fetching chapters for bookId: 85522100014
[Chapters] Total: 94, First batch: 6 chapters
[Chapters] Batch 1: Got 5 more chapters (total: 11/94)
[Chapters] Batch 2: Got 5 more chapters (total: 16/94)
...
[Chapters] ✅ Total fetched: 94/94 chapters
```

## API Response

The response will now include all episodes:

```json
{
  "success": true,
  "data": [
    { "chapterId": "...", "chapterName": "EP 1", ... },
    { "chapterId": "...", "chapterName": "EP 2", ... },
    ...
    { "chapterId": "...", "chapterName": "EP 94", ... }
  ],
  "meta": {
    "total": 94,
    "timestamp": "..."
  }
}
```

## Performance Note

- First time: Will take longer as it fetches all batches (~5-10 seconds for 100 episodes)
- Subsequent requests: Instant (served from cache for 10 minutes)
- Cache is shared across users for the same bookId

## Next Steps

**IMPORTANT:** Restart the backend server for changes to take effect:

```bash
cd dramabox-rest-api-node
npm run dev
```

Then test in your Drama Stream App - it should now load all episodes!

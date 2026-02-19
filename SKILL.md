---
name: tiktok-slideshow
description: >
  Creates and publishes TikTok slideshows via the ViralBaby API. Use when the user wants to:
  create TikTok slideshows or carousels, find/search for images for social media content,
  post or upload content to TikTok, edit slideshow text with AI, or manage image collections
  for content creation. Do NOT use for: general TikTok account management, TikTok analytics
  or metrics, video editing (this is for photo slideshows only), or any task unrelated to
  creating visual slideshow content for TikTok.
---

# ViralBaby API v1 — TikTok Slideshow Creation

Base URL: `https://viralbaby.co`

All endpoints (except auth) require: `Authorization: Bearer vb_live_...`

---

## Getting Started

### Sign Up (new user)
```
POST /api/v1/auth
Body: { "email": "user@example.com", "password": "securepassword", "action": "signup" }
Response: { "key": "vb_live_abc123...", "keyId": "uuid", "userId": "user_...", "message": "Account created..." }
```

### Log In (existing user)
```
POST /api/v1/auth
Body: { "email": "user@example.com", "password": "securepassword" }
Response: { "key": "vb_live_abc123...", "keyId": "uuid", "userId": "user_...", "message": "Logged in..." }
```

Save the `key` immediately — it is only shown once. Use it as `Authorization: Bearer vb_live_...` for all other endpoints.

---

## Image Search

### Search Unsplash
```
POST /api/v1/images/search
Body: { "query": "sunset beach", "per_page": 20, "page": 1 }
Response: {
  "searchId": "uuid",
  "results": [{
    "index": 0,
    "id": "unsplash-id",
    "description": "A sunset over the ocean",
    "thumbnailUrl": "https://...",
    "url": "https://...",
    "photographer": "John Doe"
  }],
  "total": 500
}
```
Save `searchId` — you'll need it to create collections or reference images in slideshows.

---

## Collections

### List Collections
```
GET /api/v1/collections
Response: [{ "id": "uuid", "name": "Beach Vibes", "imageCount": 12, "coverImageUrl": "..." }]
```

### Create Empty Collection
```
POST /api/v1/collections
Body: { "name": "Beach Vibes", "description": "Sunset and ocean photos" }
Response: { "id": "uuid", "name": "Beach Vibes" }
```

### Create Collection from Search Results
```
POST /api/v1/collections/from-search
Body: { "name": "Beach Vibes", "searchId": "uuid", "imageIndices": [0, 3, 5, 7] }
Response: { "collection": { "id": "uuid", "name": "Beach Vibes" }, "stats": { "total": 4, "successful": 4, "failed": 0 } }
```
This downloads the selected search result images to permanent storage.

### Get Collection Details
```
GET /api/v1/collections/{id}
Response: { "id": "uuid", "name": "...", "imageCount": 12, "images": [{ "id": "uuid", "url": "https://..." }] }
```

### Delete Collection
```
DELETE /api/v1/collections/{id}
Response: { "success": true }
```

---

## Slideshows

### List Slideshows
```
GET /api/v1/slideshows
GET /api/v1/slideshows?status=draft
Response: [{ "id": "uuid", "title": "...", "status": "draft", "slideCount": 5, "previewUrl": "https://viralbaby.co/preview/uuid" }]
```

### Create Slideshow
Each slide can get its image from three sources (in priority order):
1. `imageUrl` — direct URL to any image
2. `searchId` + `imageIndex` — reference a previous search result by index
3. `collectionId` — random image from a collection

```
POST /api/v1/slideshows
Body: {
  "title": "5 Morning Routine Tips",
  "aspectRatio": "3:4",
  "slides": [
    {
      "imageUrl": "https://example.com/photo.jpg",
      "textElements": [{ "text": "Your morning routine is broken", "type": "title" }]
    },
    {
      "searchId": "uuid",
      "imageIndex": 3,
      "textElements": [{ "text": "Here's why", "type": "title" }]
    },
    {
      "collectionId": "uuid",
      "textElements": [
        { "text": "Tip #1: Wake up early", "type": "title" },
        { "text": "Even 30 minutes makes a difference", "type": "subtitle" }
      ]
    }
  ]
}
Response: {
  "id": "uuid",
  "title": "5 Morning Routine Tips",
  "previewUrl": "https://viralbaby.co/preview/uuid",
  "slides": [...],
  "status": "draft"
}
```

Aspect ratios: `3:4` (default, best for TikTok), `1:1`, `9:16`, `4:5`

Text element types: `title` (larger, bolder), `subtitle` (smaller)
Text styles: `stroke` (default, white text with black outline), `solid` (dark background), `transparent` (shadow only)

### Get Slideshow
```
GET /api/v1/slideshows/{id}
Response: { "id": "uuid", "title": "...", "slides": [...], "status": "draft", "previewUrl": "..." }
```

### Update Slideshow
```
PUT /api/v1/slideshows/{id}
Body: { "title": "New Title", "slides": [...], "status": "ready_to_publish" }
Response: { "id": "uuid", "title": "New Title", ... }
```

### Delete Slideshow
```
DELETE /api/v1/slideshows/{id}
Response: { "success": true }
```

### Render Slides (Server-Side)
Renders slides as JPEG images with text overlaid on background images. Returns S3 URLs.

```
POST /api/v1/slideshows/{id}/render
Body: { "slideIndices": [0, 2] }   // optional, defaults to all slides
Response: { "renderedSlides": [{ "index": 0, "url": "https://s3.../rendered-slide.jpg" }] }
```

---

## AI Editing

### Edit Slide with AI
Uses AI to modify a slide's text based on natural language instructions.

```
POST /api/v1/edit
Body: {
  "slideshowId": "uuid",
  "slideIndex": 0,
  "prompt": "make the hook more punchy and add a subtitle"
}
Response: {
  "updatedSlide": {
    "textElements": [
      { "text": "Stop doing this every morning", "type": "title", "fontSize": 16, "textStyle": "stroke" },
      { "text": "it's ruining your productivity", "type": "subtitle", "fontSize": 14, "textStyle": "stroke" }
    ]
  },
  "changes": { "description": "Rewrote hook to be more direct, added subtitle" }
}
```

The edit is auto-saved to the slideshow.

---

## TikTok Integration

### Connect TikTok Account
Returns a one-time OAuth URL. Open it in any browser — no dashboard login required.

```
GET /api/v1/tiktok/connect
Response: {
  "authUrl": "https://www.tiktok.com/v2/auth/authorize/?...",
  "message": "Open this URL in any browser to connect your TikTok account."
}
```

The URL redirects through TikTok's OAuth flow and saves tokens automatically. After completing auth, you'll see a success page confirming the connection.

### Check Connection Status
```
GET /api/v1/tiktok/status
Response: { "connected": true, "tokenExpired": false, "user": { "display_name": "..." } }
// or
Response: { "connected": false, "message": "TikTok not connected..." }
```

### Upload to TikTok
Renders all slides server-side and uploads to TikTok drafts.

```
POST /api/v1/tiktok/upload
Body: { "slideshowId": "uuid", "title": "5 Tips", "description": "#productivity #morning" }
Response: { "success": true, "publishId": "tiktok-publish-id", "renderedSlides": 5 }
```

---

## Complete Workflow Example

Here's how an AI agent creates and publishes a slideshow:

```
0. Sign up or log in
   POST /api/v1/auth  { "email": "you@example.com", "password": "...", "action": "signup" }
   → save key as Authorization header

1. Search for images
   POST /api/v1/images/search  { "query": "morning routine aesthetic" }
   → save searchId

2. Create a collection from search results
   POST /api/v1/collections/from-search  { "name": "Morning Routine", "searchId": "...", "imageIndices": [0, 2, 5, 8, 11] }
   → save collectionId

3. Create the slideshow
   POST /api/v1/slideshows  {
     "title": "5 Morning Routine Tips",
     "aspectRatio": "3:4",
     "slides": [
       { "collectionId": "...", "textElements": [{ "text": "Your morning routine is broken", "type": "title" }] },
       { "collectionId": "...", "textElements": [{ "text": "Tip #1: Wake up at the same time", "type": "title" }] },
       ...
     ]
   }
   → save slideshowId, get previewUrl

4. (Optional) Edit a slide with AI
   POST /api/v1/edit  { "slideshowId": "...", "slideIndex": 0, "prompt": "make the hook shorter and more shocking" }

5. Preview the slideshow
   Open previewUrl in browser to verify

6. Connect TikTok (first time only)
   GET /api/v1/tiktok/connect  → open authUrl in browser → complete OAuth
   GET /api/v1/tiktok/status   → verify connected: true

7. Upload to TikTok
   POST /api/v1/tiktok/upload  { "slideshowId": "...", "title": "5 Morning Routine Tips", "description": "#morningroutine #productivity" }
```

---

## Error Responses

All errors follow this format:
```json
{ "error": "Human-readable error message" }
```

Common status codes:
- `400` — Bad request (missing/invalid parameters)
- `401` — Invalid or missing API key
- `404` — Resource not found
- `500` — Server error

# Feature spec — Panel image upload + Map viewer

Status: **Built (v1).** Implemented in `index.html`: the **Map** tab (coordinate layout, generation slider, lazy-loaded thumbnails, pan/zoom, full-res lightbox) and the Atlas per-panel **Upload / Replace / Remove** with per-generation art chips. Backed by the Supabase `panels` Storage bucket (private, signed URLs). The "Out of scope for v1" list at the bottom is still open. This is the long-term centerpiece: the finished panels assembled into a navigable map, viewable at any generation.

Two linked capabilities:
1. **Upload** a finished panel image (exported from Procreate, etc.) and attach it to a panel at its **current generation**.
2. **Map viewer** — all panels laid out by coordinate as thumbnails; a generation slider shows the whole map "as of" a chosen generation; click a panel for the full-res image.

---

## Prerequisites
- Supabase project live, auth + JSON world sync working (`docs/SUPABASE-SETUP.md`).
- A private Storage bucket `panels` with the RLS policies below.
- Storage uses the same `sb` Supabase client already in `index.html`.

## Decisions already made (don't re-litigate)
- **Storage backend: Supabase Storage** (not R2). Scale doesn't justify a second service; keep one stack.
- **Generation rendering: "latest ≤ G."** For a chosen generation G, a panel shows its newest image at a generation ≤ G. If it has none ≤ G, show a placeholder (the forge template / coordinate label). The map is always whole.
- **Cost discipline is a requirement, not a nice-to-have** (Supabase free tier = 1 GB storage, 5 GB egress/mo). Thumbnails in the overview; full-res only on click; lazy-load; WebP re-encode before upload.

---

## Data model

Each panel in `S.panels` gains an optional `images` map keyed by generation (string):

```js
{
  coord: "N3 E2", terrain: "ruins", founded: 5, gen: 2, order: 7,
  images: {
    "1": { path, thumbPath, w, h, bytes, uploadedAt },
    "2": { path, thumbPath, w, h, bytes, uploadedAt }
  }
}
```

- `path` / `thumbPath` are **object keys** in the `panels` bucket, not full URLs. Generate signed URLs at view time and cache them for the session.
- Backward compatible. Migration on load: `S.panels.forEach(p => p.images = p.images || {});` (add to the guards in `boot()`).
- `images` rides along in the existing JSON sync, so paths replicate across devices automatically; only the binary files live in Storage.

## Storage layout

Bucket: **`panels`** (private). Object keys:
```
{userId}/{coordNoSpace}/gen{N}.webp          // full image
{userId}/{coordNoSpace}/gen{N}.thumb.webp    // thumbnail
```
e.g. `a1b2c3d4/N3E2/gen2.webp`. `coordNoSpace` = coord with the space removed (matches the existing `nospace` form, e.g. `N3E2`).

## Storage RLS (run in Supabase SQL editor)

Create the bucket `panels` (private) in the dashboard or via SQL, then lock `storage.objects` to each user's own top-level folder (their `userId`):

```sql
create policy "own panel images: read" on storage.objects
  for select using (bucket_id = 'panels' and (storage.foldername(name))[1] = auth.uid()::text);
create policy "own panel images: insert" on storage.objects
  for insert with check (bucket_id = 'panels' and (storage.foldername(name))[1] = auth.uid()::text);
create policy "own panel images: update" on storage.objects
  for update using (bucket_id = 'panels' and (storage.foldername(name))[1] = auth.uid()::text);
create policy "own panel images: delete" on storage.objects
  for delete using (bucket_id = 'panels' and (storage.foldername(name))[1] = auth.uid()::text);
```

---

## Upload flow (client, in `index.html`)

Trigger: an **Upload** button on each Atlas panel row (next to Set current / Gen +1 / PNG). Hidden `<input type="file" accept="image/*">`.

1. Read the chosen file into an `Image`/`createImageBitmap`.
2. Render to an offscreen `<canvas>` and produce **two** WebP blobs via `canvas.toBlob(..., 'image/webp', q)`:
   - **Full:** downscale so the longest side ≤ 2400px, quality ~0.9. (Keeps storage/egress sane; finished collage PNGs are multi-MB.)
   - **Thumb:** longest side ~600px, quality ~0.8 (~60–90 KB).
3. Upload both with `sb.storage.from('panels').upload(key, blob, { upsert: true, contentType: 'image/webp' })`.
4. Record `{ path, thumbPath, w, h, bytes, uploadedAt: new Date().toISOString() }` onto `panel.images[String(panel.gen)]` (the panel's **current** generation).
5. `persist()` — JSON syncs the paths.
6. Re-render the Atlas row to show the gen now has art.

Guards:
- Only enabled when signed in (`cloudReady && sbUser`). When local-only, disable the button with a tooltip: *"Sign in (Settings → Cloud sync) to store panel images."* (Optional later: IndexedDB fallback for local-only, clearly marked as device-only / won't sync.)
- Re-uploading the same generation overwrites (`upsert`).

## Atlas UI changes

- Panel row gains an **Upload** button. After upload, show which generations have art — e.g. small chips `gen 1 ✓ · gen 2 ✓`, or a tiny thumbnail of the latest image in the row.
- Allow **Replace** (re-upload to current gen) and **Remove** (delete the gen's objects + clear `panel.images[gen]`).

## Map viewer (new tab "Map")

- Add a `Map` tab + `#v-map` view; wire into `switchView`.
- **Layout by coordinate.** Parse each coord (reuse `parseCoord`) to signed `(ew, ns)`: `x = ew` (east +), `y = -ns` (north up on screen). Place panels on a grid sized to the min/max extent. Each cell = a fixed square showing the panel's thumbnail for the selected generation (latest ≤ G) or a placeholder (coordinate label on parchment) if none.
- **Pan & zoom.** Start simple: a scrollable/translatable container with wheel-zoom and drag-pan via CSS transform. Doesn't need to be fancy v1.
- **Generation slider.** Range `1 … maxGen` (max `gen` across all panels). Default = max. On change, re-render every cell to its latest image ≤ G.
- **Click a panel** → lightbox/modal with the **full-res** image for the selected gen (signed URL), plus coord / gen / founded-year, and prev/next-generation arrows.
- **Egress discipline (required):** overview shows **thumbnails only**; full images load **only** in the lightbox on click. Lazy-load thumbnails with an `IntersectionObserver` so off-screen panels don't fetch. Cache signed URLs in a session map.

## Serving URLs

- Private bucket → `sb.storage.from('panels').createSignedUrl(path, 3600)` per image; cache results for the session to avoid re-signing.
- (Alternative if you ever want a public, shareable map: make the bucket public and use `getPublicUrl` — simpler and CDN-cacheable, but anyone with a link can view. Default to **private + signed** unless the user asks for a public share.)

## Acceptance criteria

- A signed-in user can upload a finished panel image; it attaches to the panel's current generation, survives reload, and appears on a second device.
- The Atlas shows which generations of a panel have art, and supports replace/remove.
- The Map tab renders all panels positioned by coordinate, thumbnails only, lazy-loaded.
- The generation slider re-renders the whole map using "latest ≤ G"; the map is never broken/empty for panels that exist.
- Clicking a panel opens the full-res image for the chosen generation.
- Everything degrades gracefully when not signed in (upload disabled with a hint; viewer shows placeholders for any panels lacking images).

## Out of scope for v1 (note for later)
- IndexedDB local fallback for unsigned-in image storage.
- Exporting the whole assembled map as one composited image.
- Public read-only share links per generation.
- Multiple named worlds per account (would also touch the base schema).

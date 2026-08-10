# MyINDAH Diet — API Collection

Bruno collection covering the existing endpoints and the new features being built for Paket A (Esensial).

Maintainers:
- **Backend** — Muhammad Zaid Taufiq Yasyaf <zaid.ug@gmail.com>
- **Mobile (Flutter)** — Bagas Dewanggono <bagasdewa07@gmail.com>

## Quickstart

1. Install Bruno: <https://www.usebruno.com/downloads>
2. Open Bruno → **Open Collection** → point at this folder
3. Top-right environment dropdown → select **Local** (for `http://127.0.0.1:8000`) or **Production** (for `https://webapi.myindah-app.com`)
4. Run **Auth/LOGIN** with the default credentials (`admin@gmail.com` / `password`). The response token is auto-saved to the `token` env var.
5. All other requests inherit the bearer token from the collection-level auth — no manual copy-paste needed.

To use a different login, edit the `email`/`password` vars in `collection.bru` (top-level pre-request vars) before hitting LOGIN.

## Folder Map

| Folder | Purpose |
| --- | --- |
| `Auth/` | `LOGIN`, `LOGOUT`, `REFRESH` (new — exchanges current JWT for a fresh one before expiry) |
| `Register dan Upload Data User/` | Self-service user `REGISTER`, `VIEW` (current user), `UPDATE` profile, `UPDATE FCM TOKEN` (new — push notification registration) |
| `Article/` | Existing public/mobile article (resep blog) endpoints |
| `Farm Record/` | **NEW** Petani-side endpoints: start padi cycle, list activities, counts, add custom activity, mark done |
| `Harga Pangan/` | **NEW** Konsumen-side endpoints: today's prices + filter meta |
| `Resep Rekomendasi/` | **NEW** Konsumen-side weather-based recipe recommendation, listing, detail |
| `Admin - Recipes/` | **NEW** Recipe CRUD for the admin panel (BRIN populates kcal, durasi, photos, instructions) |
| `Admin - Recipe Recommendations/` | **NEW** Linking a recipe to (region × weather × meal_time) |
| `Admin - Harga Pangan/` | **NEW** Snapshot audit + manual scrape trigger |
| `Admin - Farm Record/` | **NEW** Read-only audit of cycles & activities |

## Conventions

- All write requests use **multipart-form** bodies (not JSON) — matches the existing app convention. Bruno fills the multipart Content-Type automatically; do not set it manually.
- Updates that may include a file use `POST` with a `_method=PATCH` form field (Laravel multipart hack).
- Response envelope is consistent across endpoints:
  ```json
  { "success": true, "message": "Human-readable status", "data": <payload> }
  ```
  except for `LOGIN` which keeps the legacy `{ success, user, permissions, token }` shape for backward compatibility with the existing app.
- Errors return HTTP 4xx with the same envelope shape:
  ```json
  { "success": false, "message": "Email or Password is incorrect" }
  ```
  or validation errors as the raw Laravel validation map (HTTP 422):
  ```json
  { "email": ["The email has already been taken."] }
  ```

## Auth Model — two-token (OAuth2-style)

LOGIN returns BOTH:
- `token` — JWT access token, TTL **60 minutes**. Sent as `Authorization: Bearer <token>` on every API call.
- `refresh_token` — opaque random string, TTL **14 days**. Stored server-side (hashed). Sent only to `/auth/refresh`.

**Refresh flow:**
1. Mobile client calls `LOGIN`, stores both tokens securely.
2. When an authenticated request returns 401, client POSTs the stored `refresh_token` to `/auth/refresh`.
3. Server validates the refresh token, **revokes it (rotation)**, returns a NEW pair (`token` + new `refresh_token`).
4. Client replaces BOTH stored tokens with the new pair, then retries the original request.
5. If refresh itself returns 401 (revoked / expired / unknown), the user must log in again.

**Logout** invalidates both: the current JWT is added to the server blacklist, and the refresh token (if sent in body) is revoked in DB. Send both for clean logout.

## Environment Variables

Two environment files live in `environments/`:

- `Local.bru` → `baseUrl = http://127.0.0.1:8000/api`
- `Production.bru` → `baseUrl = https://myindah-api.toko.center/api`

The `token` and `refresh_token` vars are filled automatically by the post-response scripts on `LOGIN` and `REFRESH`. Other ad-hoc vars (`userId`, `recipeId`, `activityId`, `slug`, etc.) live as per-request pre-request vars and can be overridden in the request panel without editing files.

## Notes for the Flutter Dev

- Sample response bodies are embedded in each request's `docs` block so you can build models without waiting for the live endpoint.
- All slugs (`provinsi`, `jenis_pasar`, `komoditas`, `weather_slug`, `meal_time_slug`, `region_slug`, `city`) are stable kebab-case identifiers; use them in API calls rather than display labels.
- For Harga Pangan, the `GET /mobileapps/harga-pangan/meta` endpoint returns the full list of valid slugs + display labels for dropdown rendering.

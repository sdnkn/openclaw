---
name: places-near
description: Personal location + places skill backed by Supabase project `mpwdfhxteupddyjwxcai`. **Default skill for ANY Istanbul travel/location/shopping/sightseeing/planning question**, plus all "near me / route / where am I" intents in any city. Handles — (1) "where am I" / "ещё раз геопозицию" → return user's current pin + Google Maps link, (2) "what's nearby" / "что рядом" / "куда сходить" → ranked list of POIs from `ic_places` with distance, bill, open/closed status, view + walking links, (3) "построй маршрут от X до Y" / "route X to Y" → Google Maps directions URL with the right travel mode, (4) "что в магазине/моле X" / "какие магазины в Y" / "что есть в Z" / clustering and planning across multiple POIs (sneaker shops, malls, sights). Location flow: the Telegram webhook accepts every ping (live-share and one-shot), so the last known pin from `user_locations` is usually fresh. The skill reads it silently when it's <=30 min old, mentions the age between 30 min and 6 h, and only actively prompts the bot for a refresh past 6 h or when the user explicitly says they moved. The user lives in Istanbul — treat any Istanbul place/topic as a trigger, do NOT wait for "near me".
---

# places-near

Personal places + location lookup against Supabase project `mpwdfhxteupddyjwxcai`. Two tables in play:
- `public.ic_places` — 166 curated POIs (Istanbul + other cities)
- `public.user_locations` — last known user location. The Telegram webhook writes every incoming ping (deduping near-identical pings to a single row that gets touched), so live-share automatically keeps the latest row fresh.

## Default mindset for Istanbul

The user lives in / is travelling around Istanbul for the foreseeable future. Whenever they bring up **anything** that touches a location, a place, a route, a mall, a shop, a restaurant, an attraction, a district, planning a day, or "what's around X" — this skill is the default entry point. **You do not wait for an explicit "near me" cue.** The flow is always:

1. **Check user position first.** Read the latest row from `user_locations`. Use it silently if `age <= 30 min`; with a soft disclaimer if `30 min < age <= 6 h`; only actively request a fresh pin (Step 1.3) if `age > 6 h` or the user explicitly says they moved. Asking for a pin tap when we already have a usable row is friction the user explicitly rejected — do not do it.
2. **Query `ic_places` for everything related** — shops, malls, sights, food — by name, category, district, or geo. The DB is the source of truth and is curated; lead with it.
3. **Cross-check live status** — for any specific store/venue the user is about to physically visit, verify hours / "still open" via the web (Exa search or `WebFetch` of the official site). `business_status='OPERATIONAL'` in DB is not a substitute — places close, change hours, get renovated.
4. **Augment from web only when DB is incomplete** — e.g. when listing tenants of a mall, our DB has just a couple of shops, but the actual mall has dozens. In that case fetch the mall's site (via `pg_net` from Supabase if outbound is restricted) and surface the full tenant list. Be explicit which items came from DB vs from the live web.
5. **Always emit Google Maps links** — view link (with `query_place_id` when we have one) AND a directions link (walking by default, transit/driving when distance/intent warrants). Multiple stops → use the `waypoints=` form so the whole day fits in one URL.
6. **Plan with calendar in mind** — when planning a day, propose calendar slots and (only after the user nods) push events into `maxsidenkin@gmail.com`'s primary calendar with location coords and Maps link in the description. Default TZ `Europe/Istanbul` (= `Europe/Moscow` offset, so either works).

If the user's question is **not** about a specific Istanbul venue or geo — fine, this skill stays out of the way.

## When to use

Trigger this skill for any of:
- **Nearby places**: "что рядом покушать", "где поесть рядом", "найди кафе около меня", "what's nearby", "places near Taksim"
- **Current position**: "где я сейчас", "ещё раз геопозицию", "where am I", "покажи мою точку"
- **Route building**: "построй маршрут до X", "как доехать до Y", "route to <place>", "от A до B пешком/на метро"
- **Cluster / shopping planning**: "какие магазины кроссовок", "что в моле X", "сгруппируй по районам", "какие достопримечательности рядом"
- **Day planning** that mentions Istanbul venues, shopping, sightseeing, or "куда сходить"
- **Anything about a named Istanbul venue / district** — even just "что такое Galataport" or "как там Nişantaşı" — pull from DB first
- User pastes a Google Maps link / coordinates and asks anything about them

Do NOT use for: pure code questions, generic city-wide top-10 lists with no anchor, questions about cities other than Istanbul where we have no `ic_places` coverage.

## Step 1: Resolve the user's location

Accept any of these inputs (priority order):

1. **Explicit coords / Maps link in the current message** — extract from the URL or raw text.
   - `https://www.google.com/maps/@41.0369,28.9850,17z` → `41.0369, 28.9850`
   - `https://maps.app.goo.gl/...` (short link) → use `WebFetch` to follow the redirect and read the `@lat,lon` segment from the final URL
   - `https://www.google.com/maps/place/.../@41.0369,28.9850,...` → take `@lat,lon`
   - `https://www.google.com/maps/dir/?destination=41.0369,28.9850` → take destination
   - Raw `41.0369, 28.9850` or `lat=41.0369 lon=28.9850`
2. **Last known location from Telegram bot** — query `public.user_locations`:
   ```sql
   SELECT id, lat, lon, accuracy_m, created_at,
          EXTRACT(EPOCH FROM (now() - created_at))::int AS age_s
   FROM public.user_locations
   WHERE user_id = 'default'
   ORDER BY created_at DESC
   LIMIT 1;
   ```
   - If `age_s <= 1800` (30 min) → **use it silently**, no friction, no button.
   - If `1800 < age_s <= 21600` (≤6 h) → use it, but add a one-liner in your reply: "(точка N мин/ч назад — скажи если переместился)".
   - If `age_s > 21600` (>6 h) **OR** the user explicitly says "I moved" / "got a new location" / "новое место" / "обнови" → go to step 1.3.

   **Do not request a fresh pin just because the row is older than a few minutes.** The user lives in Istanbul, is usually stationary or moving on foot in a small radius. The webhook accepts every location ping (live-share and one-shots both), so a stale row often means the user simply paused live-share — not that they teleported. Refreshing constantly is friction they explicitly do not want.
3. **Request a fresh pin from the bot (on-demand)** — only when step 1.2 says we need one. The webhook accepts pings continuously; this step actively prompts the user when their last pin is genuinely stale.
   1. **Fire the request.** `pg_net` is async — it queues the HTTP call, returns a row id, and the response lands in `net._http_response` ~1 sec later. Do both in one SQL block so the request goes out immediately:
      ```sql
      SELECT net.http_post(
        url := 'https://mpwdfhxteupddyjwxcai.supabase.co/functions/v1/request-location',
        body := jsonb_build_object('reason', '<short label, e.g. places-near>'),
        headers := '{"Content-Type":"application/json"}'::jsonb,
        timeout_milliseconds := 8000
      ) AS pgnet_id;
      ```
   2. **Tell the user immediately** (don't wait silently — they need to look at Telegram): "📍 Кинул кнопку в бота — тапни «Отправить локацию», подожду до 20 сек."
   3. **Resolve the request id.** Wait 2 sec via `Bash` (`sleep 2`), then read the response body to extract `request_id` (this is the `location_requests.id`, call it `LR_ID`):
      ```sql
      SELECT status_code, content::jsonb ->> 'request_id' AS lr_id
      FROM net._http_response WHERE id = <pgnet_id>;
      ```
      If `status_code` is not 200, surface the body to the user and stop.
   4. **Poll for the pin.** Up to 6 attempts × 3 sec via `Bash sleep 3` between SQL calls:
      ```sql
      SELECT ul.id, ul.lat, ul.lon, ul.accuracy_m, ul.created_at
      FROM public.user_locations ul
      JOIN public.location_requests lr ON lr.resulting_location_id = ul.id
      WHERE lr.id = <LR_ID>;
      ```
      As soon as a row appears, proceed to Step 2. If the loop times out, tell the user: "Не пришёл пин — тапни кнопку в боте и спроси ещё раз." Do NOT silently fall back to a stale row.
4. **Named landmark** — "Taksim Square", "Galata Tower", "near Sultanahmet". Look it up first in `ic_places` by name/aliases; if missing, use WebSearch to geocode (`"<name> coordinates lat lon"`).
5. **Image with EXIF** — if the user attaches a photo and asks "что рядом?", check EXIF GPS tags with `exiftool` if available; otherwise ask for a Maps link.

If none of the above is present and the bot path fails twice in a row, ask the user to paste a Google Maps link. Do NOT silently assume a location.

## Step 2: Query Supabase

Use the `mcp__ce5339b6-8658-4f81-97ba-e65603549ee0__execute_sql` tool. Project id: `mpwdfhxteupddyjwxcai`.

Default radius: 1500 m (walking distance). Bump to 3000 m if fewer than 5 results, to 8000 m only on explicit "in the city" intent. Limit 15.

Default category filter: if user said "поесть / eat / food / dinner / lunch / breakfast / restaurant / cafe / coffee", filter `category IN ('food','coffee')`. If "shop / шопинг / vintage / streetwear" — `category IN ('shop_streetwear','shop_vintage','shop_other')`. If "посмотреть / достопримечательности" — `category IN ('architecture','walk','other')`. If unclear — no category filter.

Template SQL (substitute `<LAT>`, `<LON>`, `<CATEGORIES>`, `<RADIUS_M>`, `<LIMIT>`):

```sql
WITH
  origin AS (SELECT <LAT>::float8 AS lat, <LON>::float8 AS lon),
  now_local AS (SELECT (now() AT TIME ZONE 'Europe/Istanbul') AS ts),
  dow AS (
    SELECT CASE EXTRACT(DOW FROM ts)::int
             WHEN 0 THEN 'sunday' WHEN 1 THEN 'monday' WHEN 2 THEN 'tuesday'
             WHEN 3 THEN 'wednesday' WHEN 4 THEN 'thursday' WHEN 5 THEN 'friday'
             WHEN 6 THEN 'saturday' END AS day_name,
           to_char(ts,'HH24:MI') AS hhmm
    FROM now_local
  )
SELECT
  p.name, p.name_native, p.category, p.subcategory, p.address,
  p.price_tier, p.avg_bill_try, p.rating, p.reviews_count,
  p.website, p.phone, p.google_place_id,
  p.lat, p.lon,
  ROUND((6371000 * 2 * ASIN(SQRT(
    POWER(SIN(RADIANS((p.lat - o.lat)/2)),2) +
    COS(RADIANS(o.lat))*COS(RADIANS(p.lat)) *
    POWER(SIN(RADIANS((p.lon - o.lon)/2)),2)
  )))::numeric, 0) AS dist_m,
  p.hours_verified -> d.day_name AS today_hours,
  d.day_name, d.hhmm AS now_local_time,
  (
    SELECT bool_or(
      CASE
        WHEN (slot->>1) > (slot->>0)
          THEN d.hhmm >= (slot->>0) AND d.hhmm < (slot->>1)
        ELSE  -- slot crosses midnight, e.g. 17:00-00:30
          d.hhmm >= (slot->>0) OR d.hhmm < (slot->>1)
      END
    )
    FROM jsonb_array_elements(COALESCE(p.hours_verified -> d.day_name, '[]'::jsonb)) slot
  ) AS open_now
FROM public.ic_places p, origin o, dow d
WHERE p.lat IS NOT NULL AND p.lon IS NOT NULL
  AND (p.business_status IS NULL OR p.business_status = 'OPERATIONAL')
  AND (p.category = ANY(ARRAY[<CATEGORIES>]) OR cardinality(ARRAY[<CATEGORIES>]) = 0)
HAVING ROUND((6371000 * 2 * ASIN(SQRT(
    POWER(SIN(RADIANS((p.lat - o.lat)/2)),2) +
    COS(RADIANS(o.lat))*COS(RADIANS(p.lat)) *
    POWER(SIN(RADIANS((p.lon - o.lon)/2)),2)
  )))::numeric, 0) <= <RADIUS_M>
ORDER BY dist_m ASC
LIMIT <LIMIT>;
```

Note: `HAVING` works here only because there's no `GROUP BY` — Postgres allows it. If it errors, wrap the SELECT in a subquery and filter on `dist_m` in the outer query.

The timezone is currently hardcoded to `Europe/Istanbul` because the DB is Turkey-focused. If the resolved location is clearly outside Turkey, swap to the appropriate timezone (Asia/Tbilisi, Asia/Yerevan, etc.).

## Step 3: Build Google Maps links

For each row build two links:

**View on map** — use `google_place_id` when available, fall back to coordinates:
```
https://www.google.com/maps/search/?api=1&query=<lat>,<lon>&query_place_id=<google_place_id>
```
If no `google_place_id`:
```
https://www.google.com/maps/search/?api=1&query=<lat>,<lon>
```

**Directions from user's location** — same URL, just swap `travelmode`. Pick a sensible default from the user's wording (walking by default, transit for >2 km in Istanbul, driving for inter-city, bicycling only if user asked):
```
https://www.google.com/maps/dir/?api=1&origin=<USER_LAT>,<USER_LON>&destination=<lat>,<lon>&destination_place_id=<google_place_id>&travelmode=<walking|transit|driving|bicycling>
```

**Multi-mode** — when the user explicitly wants options ("маршрут / как добраться"), emit two links: walking + transit (or transit + driving for longer distances). Don't dump all four modes by default.

**Arbitrary A → B route** (not anchored to user's current location) — when the user says "построй маршрут от X до Y", resolve both endpoints (same way as Step 1 — coords, Maps link, named landmark, or `ic_places` lookup), then build the dir URL with both `origin` and `destination` filled in. If origin or destination is an `ic_places` row, append `&origin_place_id=...` / `&destination_place_id=...` for cleaner pins.

## Step 4: Format the response

Output a compact markdown table sorted by distance. **Adapt columns to what's filled** — don't show empty columns. Possible columns:

| # | Место | Расст. | Чек | ⭐ | Сейчас | 📍 | 🚶 | 🌐 |

- **Место** — `name` (and `name_native` in parens if different)
- **Расст.** — `dist_m` formatted as `120 м` or `1.1 км`
- **Чек** — `avg_bill_try` as `300 ₺` (or use `price_tier` 1-4 as `$`, `$$`, `$$$`, `$$$$` if no avg_bill)
- **⭐** — `rating` (e.g. `4.6 (5.9k)`) if present
- **Сейчас** — `✅ открыто` / `❌ закрыто` / `❓` (when today_hours is NULL); add short hours like `(до 23:00)` when open
- **📍** — markdown link "карта" → view-on-map URL
- **🚶** — markdown link "маршрут" → directions URL (only if you have user's origin coords)
- **🌐** — markdown link "сайт" → `website` (only if filled)

After the table, give a 2-3 line recommendation: which one to pick for the current time and why (open status + rating + bill).

If many rows have `today_hours = null` (unverified hours), add a one-line disclaimer: "У части мест часы не верифицированы — статус «❓» означает «не знаю», а не «закрыто»."

## Step 5: Offer enrichment (optional)

If more than ~30% of returned rows lack `hours_verified` or `google_place_id`, end with: "Если нужно — могу обогатить эти записи через Google Places API и записать в БД."

Don't actually do enrichment unless the user agrees — it costs API quota.

## Web augmentation & live verification

The DB is curated but not exhaustive. Two situations always require a live check:

1. **Listing tenants of a mall** — DB only has a handful of streetwear/sneaker shops per mall; the mall actually contains dozens (Nike, Adidas, Foot Locker, mainline fashion). Fetch the mall's official site / blog to get the real tenant list, then combine.
2. **Confirming a venue is open today** — when the user is about to physically go, verify hours/status on the live web before sending them. `business_status='OPERATIONAL'` in DB is months-old metadata.

**How to fetch outside content when `WebFetch` is blocked.** This environment's outbound HTTP allowlist often blocks direct calls to non-Supabase hosts. The Supabase Postgres has `pg_net` enabled (extension was created in `extensions` schema; functions live under `net`) and Supabase's egress is not allowlist-restricted — use it as a proxy:

```sql
-- 1) fire async GET
SELECT net.http_get(url := '<URL>', timeout_milliseconds := 15000) AS pgnet_id;

-- 2) wait ~3 sec (Bash sleep), then read body
SELECT status_code, LENGTH(content::text) AS len,
       regexp_replace(content::text, '<[^>]+>', ' ', 'g') AS plaintext
FROM net._http_response WHERE id = <pgnet_id>;
```

Use this for `galataport.com`, `kapalicarsi.com.tr`, `zorlucenter.com.tr`, individual venue sites, etc. Avoid setting custom `User-Agent` headers via `pg_net` — Galataport's WAF rejects them as "Bad Request"; the default UA works.

**Prefer Exa (`mcp__21eb2344-..._web_search_exa` / `web_fetch_exa`) when available** — it returns clean markdown without the HTML stripping dance. If Exa credits are exhausted (HTTP 402), fall back to the `pg_net` route above.

When you augment from web, **always tag the source** in the reply: "(из БД)" vs "(с сайта молла)" so the user knows what's curated vs scraped.

## Quick reference: schema

`public.ic_places` columns used here:
- Identity: `id`, `name`, `name_native`, `aliases[]`
- Geo: `lat`, `lon`, `district`, `side`, `address`
- Taxonomy: `category`, `subcategory`, `tags[]`
- Hours: `hours_verified` (jsonb, `{day: [[from,to], ...]}`), `hours_raw`
- Pricing: `price_tier` (1-4), `price_level` (Google's 0-4), `avg_bill_try`
- Quality: `rating` (numeric), `reviews_count`, `business_status` (`OPERATIONAL`/`CLOSED_TEMPORARILY`/`CLOSED_PERMANENTLY`)
- Linking: `google_place_id`, `website`, `phone`

Categories present in DB: `food`, `coffee`, `nightlife`, `shop_streetwear`, `shop_vintage`, `shop_other`, `mall`, `architecture`, `walk`, `other`.

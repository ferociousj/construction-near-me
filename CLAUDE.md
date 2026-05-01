# Construction Near Me

## What This Is
A website that lets people find and report active construction sites on a map. Target audience is parents of small kids who want to watch big trucks, plus community members who help by reporting sightings. Brainchild of Jon Sung (https://linktr.ee/ferociousj).

## Live Site
- URL: https://ferociousj.github.io/construction-near-me/
- Debug mode: https://ferociousj.github.io/construction-near-me/?debug=true
- Hosted on GitHub Pages. Single `index.html` file — the entire app lives in one file.
- GitHub repo: https://github.com/ferociousj/construction-near-me

## Tech Stack
- **Frontend:** Vanilla HTML/CSS/JS (no framework, no build step)
- **Map:** Leaflet + OpenStreetMap tiles (free, no API key)
- **Geocoding:** Nominatim (OpenStreetMap's geocoder, free, no API key)
- **Permit Data:** Socrata (SODA) APIs from multiple city open data portals (free, no API key)
- **Database:** Supabase (optional — works in offline/localStorage mode without it)
- **Hosting:** GitHub Pages (free)

## Architecture Decisions

### Single-file app
Everything is in one `index.html` for maximum simplicity. The creator is not a developer, so minimizing files, build tools, and complexity is a priority.

### Supabase integration is optional
The app detects whether Supabase credentials are present (hardcoded or in localStorage). If yes, it connects to the shared database. If not, it runs in offline mode where each visitor has their own pins in localStorage.

### Hardcoded credentials
Supabase URL and anon key can be hardcoded in two clearly labeled constants at the top of the script section:
```javascript
const HARDCODED_URL = '';
const HARDCODED_KEY = '';
```
The boot() function checks these with .trim() and .length > 0 to avoid whitespace/null issues. If filled in, the setup screen never appears.

### Security model (server-side enforcement)
- **Row Level Security (RLS)** on Supabase enforces all access rules server-side
- **Visitors** can: read all pins (`SELECT`), refresh a pin's date via "Still there?" (`refresh_pin` RPC), update machines/notes via "Update details?" (`update_pin_report` RPC), and add new sightings via `create_pin` RPC
- **Admin only** can: update pins (`UPDATE`), delete pins (`DELETE`), modify settings
- **Direct INSERT is blocked** — there is no public `INSERT` policy on the pins table. All inserts go through the `create_pin()` function.
- Admin is authenticated via a **secret token** stored in the `settings` table (`admin_token` key)
- The token is passed as a custom `x-admin-token` header on protected operations
- Supabase's `is_admin()` database function checks the token server-side before allowing the operation

### Database functions (SECURITY DEFINER)
These bypass RLS but are tightly scoped:
- **`create_pin(p_lat, p_lng, p_machines, p_good_view, p_date_reported, p_custom_machine, p_notes)`** — the only way to add a pin. Enforces a rate limit of 10 inserts per minute (global) and a cap of 500 total pins. Truncates notes to 140 characters server-side. Returns the new pin's `id` as JSON. These limits can be adjusted by editing the function in the Supabase SQL Editor.
- **`refresh_pin(pin_id)`** — updates only `date_reported` to `now()` for a single pin. Used by the "Still there?" button. Anyone can call it.
- **`update_pin_report(p_pin_id, p_machines, p_custom_machine, p_notes)`** — updates machines, custom_machine, notes, and resets `date_reported` to `now()`. Used by the "Update details?" flow after confirming "Still there?". Rate-limited to 10 updates per minute. Truncates notes to 140 characters server-side. Anyone can call it.
- **`is_admin()`** — checks if the `x-admin-token` request header matches the `admin_token` value in the `settings` table. Used by RLS policies for UPDATE/DELETE.

### What an attacker with the anon key can do
- Read pins — fine, it's public data
- Insert pins — only through `create_pin()`, max 10/minute, max 500 total
- Update `date_reported` — only via `refresh_pin()`, one pin at a time
- Update machines/notes — only via `update_pin_report()`, rate-limited to 10/minute, notes truncated to 140 chars
- Delete pins — blocked without admin token
- Modify settings — blocked without admin token

### Security-related files
- `RLS-UPGRADE.sql` — sets up RLS policies, admin token, `is_admin()`, and `refresh_pin()`
- `INSERT-LOCKDOWN.sql` — removes public INSERT policy, creates `create_pin()` with rate limiting
- `ADD-NOTES-FIELD.sql` — adds `notes` column, drops old `create_pin()` and recreates with `p_notes` parameter
- `UPDATE-DETAILS.sql` — creates `update_pin_report()` for the "Update details?" feature

### Leaflet popup state management
Pin popups use a **pre-rendered three-state approach**: all three UI states (default "Still there?" button, confirmed message with "Update details?" link, and the full edit form) are rendered in the popup HTML upfront with `display:none`. Button clicks toggle visibility between states. This avoids `innerHTML` mutation inside Leaflet popups, which causes popups to close or break due to Leaflet's internal DOM event handling. The popup also uses `closeOnClick: false` to prevent Leaflet from closing it during button interactions.

### Admin interface
- Accessed via a barely-visible "admin" text link in the top-right of the header
- Admin logs in with a **token** (generated by the SQL migration, stored in browser localStorage after first entry)
- Features: sortable/filterable table (includes notes column), bulk delete, bulk set freshness, change password
- "Change password" requires matching entries; mismatch shows "Your entries don't match, friend"

## Data Model

### Pins table (Supabase)
```sql
id UUID (auto-generated)
lat DOUBLE PRECISION
lng DOUBLE PRECISION
machines TEXT[]
good_view BOOLEAN
date_reported TIMESTAMPTZ
custom_machine TEXT
notes TEXT (max 140 chars, enforced server-side)
```

### Settings table (Supabase)
```sql
key TEXT (primary key)
value TEXT
```
Stores `admin_password` (legacy, used for the change-password UI) and `admin_token` (the actual secret used for RLS enforcement).

### Pin freshness & color
- Freshness = days since date_reported
- Green: ≤ 7 days
- Yellow: 8–14 days  
- Red: > 14 days
- Default filter shows pins ≤ 30 days old

### Machinery options (checkboxes)
Bulldozer, Excavator, Backhoe, Front Loader, Bobcat/Skid Steer, Concrete Mixer, Crane Truck, Construction Crane, Dump Truck, Concrete Pumper, Wrecking Ball, Asphalt Spreader, Road Grader, Steamroller, Forklift, plus a freeform text field.

## Features Implemented
- Interactive Leaflet map with OpenStreetMap tiles
- Color-coded pins by freshness (green/yellow/red)
- "Report a Sighting" — click button, click map, fill overlay, save
- Pin detail popups with machinery tags, good view flag, date, notes, "Still there?" button
- "Still there?" resets date_reported to now (resets freshness), then offers "Update details?" link
- "Update details?" — inline editing of machines, custom machine, and notes within the pin popup; also re-refreshes date on save
- Notes field on sightings — 140-character freeform text with live character counter, displayed in pin popups and admin table
- Search & Filter panel: location search (Nominatim geocoding), freshness range, good view filter, machinery filter
- Admin interface: sortable/filterable table, select all/deselect all, bulk delete, bulk set freshness, change password
- About overlay with attribution to Jon Sung
- Supabase integration with offline fallback
- Supabase credentials hardcoded — setup screen is skipped for all visitors
- Server-side Row Level Security with admin token authentication
- "Still there?" uses `refresh_pin` RPC function — works for all visitors without admin access
- Sample pins around San Francisco pre-loaded
- Multi-city permit data layer with 6 working cities (see below)
- Count badge (top-right) shows both user reports and permit counts
- Debug mode via `?debug=true` URL parameter (see Debugging section)

## Multi-City Permit Data System

### How it works
A city selector dropdown in the Search & Filter panel lets users switch between cities. Selecting a city pans the map to that city's center and fetches permit data from that city's open data API. Permits display as blue square pins, distinct from the round colored user-report pins.

### City config architecture
Each city is defined by a config object (`CITY_CONFIGS`) containing:
- `name` — display name
- `center` — [lat, lng] for map centering
- `api` — SODA API endpoint URL
- `costField` — which field holds the project cost (or null if none)
- `costIsText` — whether the cost field needs `::number` casting in SoQL
- `statusFilter` — SoQL filter for issued/active permits (or null to skip)
- `dateField` — which field holds the issue date (or null to skip date filtering)
- `locationMode` — `'object'` (location object field) or `'latlong'` (separate lat/lng fields)
- `locationField` — custom name for the location object field (default: `'location'`)
- `latField` / `lngField` — field names for lat/lng when using latlong mode
- `latFallback` / `lngFallback` — backup lat/lng fields if primary location parsing fails
- `select` — comma-separated field names for the $select clause
- `parse` — function to normalize a raw API row into the standard permit object

Adding a new city is just adding a new config block and a dropdown option. No other code changes needed.

### Cities — all confirmed working ✅
| City | API Domain | Dataset ID | Cost Slider | Notes |
|------|-----------|------------|-------------|-------|
| San Francisco | data.sfgov.org | i98e-djp9 | ✅ `estimated_cost` (text, cast) | Location is an object field with GeoJSON coordinates. |
| Chicago | data.cityofchicago.org | ydr8-5enu | ✅ `reported_cost` (text) | Status filter: `permit_status='ACTIVE'`. |
| New York City | data.cityofnewyork.us | ipu4-2q9a | ❌ No cost field | Date fields use MM/DD/YYYY strings — `dateField` set to null, skips date filter. Orders by `:id DESC`. |
| Los Angeles | data.lacity.org | pi9x-tg5x | ✅ `valuation` (text, cast) | Slow API (~13 seconds). Uses `geolocation` object field with `lat`/`lon` text fallback. No status filter. Dataset is "2020 to Present". |
| Seattle | data.seattle.gov | 76t5-zqzr | ✅ `estprojectcost` (text) | No status filter — filters by `issueddate` existence. |
| Austin | data.austintexas.gov | 3syk-w9eu | ❌ No cost field | Status filter: `status_current='Active'`. Date field is `issue_date`. |

### Cities removed (APIs not accessible or unusable)
| City | Reason |
|------|--------|
| Washington DC | `opendata.dc.gov` blocks browser fetch requests (CORS). Would need a proxy server. |
| Portland | ArcGIS endpoint returned 404 "Service not found". The service URL has changed or been retired. |
| San Diego County | `geocoded_column` field is mostly empty — permits have addresses but no usable coordinates for mapping. |
| Nashville | `data.nashville.gov` blocks browser fetch requests (CORS). |
| Minneapolis | `opendata.minneapolismn.gov` blocks browser fetch requests (CORS). |
| Honolulu | Dataset ID `mhcf-stc5` not found (404). Correct dataset needs research. |
| Miami-Dade | ArcGIS Hub endpoint blocks browser fetch requests (CORS). |
| Philadelphia | Dataset `ds7g-p84b` is business certifications, not permits. Correct permit dataset needs research. |
| Boston | Uses CKAN (not Socrata) — would need a different query format. |
| Denver | No Socrata building permit dataset found. |

### Cost filter
- Logarithmic slider from $0 to $500M, default $5M
- Slider position 0–1000 maps to cost via: `10^(sliderVal/1000 * log10(500000000))`
- "Reload Permits" button re-fetches data with the current cost threshold
- Cost filter uses a 3-step fallback per city:
  1. Try with `::number` cast (for text fields like SF)
  2. Try without cast (for native number fields)
  3. Try without cost filter at all (guaranteed to work if field names are correct)
- Cities with no cost field (NYC, Austin) ignore the slider entirely
- TODO: Show "Cost filter not available for this city" when a no-cost city is selected

### Location parsing
The `extractLatLng()` function handles multiple formats:
- `{ latitude: "37.7", longitude: "-122.4" }` — object with string lat/lng
- `{ type: "Point", coordinates: [-122.4, 37.7] }` — GeoJSON format
- `"POINT (-122.43 37.75)"` — WKT string format
- Separate `lat`/`lon` text fields via `latFallback`/`lngFallback` (used by LA)

### API resilience
- All fetch calls have a 15-second timeout via AbortController
- The "Reload Permits" button uses try/finally to always reset from "Loading..." state
- When `dateField` is null, the date filter is skipped and ordering falls back to `:id DESC`
- When `statusFilter` is null, no status filtering is applied

### Debugging
A comprehensive debug mode is available at `?debug=true`. It replaces the page with a plain-text report that runs 6 tests for every configured city:
1. **Raw fetch** — 2 rows with no filters to see actual field names
2. **Status field check** — queries for distinct status values
3. **Date field check** — confirms date field exists, shows most recent value
4. **Cost field check** — checks field existence, samples values, tests both `::number` cast and direct comparison to determine type
5. **Location field check** — confirms lat/lng or location object fields exist, shows sample coordinates
6. **Full query** — runs the actual app query, shows parsed results

The debug report can be read by fetching the live URL. To share with Claude for debugging, paste the text output.

**TODO:** Add a password gate to the debug mode so it's not publicly discoverable (currently accessible to anyone who knows the URL parameter).

## Features NOT Yet Implemented
- Directions to a pin from user's location (noted as nice-to-have in design doc)
- "Cost filter not available" message for NYC and Austin
- Password-protected debug mode

## Future Options (discussed but not built yet)

### Google Maps
Swap Leaflet/OSM for Google Maps JavaScript API. Requires Google Cloud account, API key, and billing info (free tier covers ~28,000 map loads/month). Would also swap Nominatim for Google Geocoding API. Ready to implement when requested.

### Custom domain
Cloudflare Registrar recommended as cheapest option (~$10-11/year for .com, sold at wholesale cost). `constructionnear.me` noted as a clever option. GitHub Pages supports custom domains — just needs a CNAME file and DNS config.

### ~~Stronger security~~ ✅ DONE
Row Level Security is now enforced server-side with admin token authentication. See "Security model" section above.

### More cities
To add a new city:
1. Find its open data portal
2. Identify the building permits dataset
3. Run `?debug=true` or download a few rows as CSV to check field names
4. Create a config block in `CITY_CONFIGS`
5. Add an `<option>` to the city selector dropdown in the HTML

Many cities were tried and failed due to CORS, missing datasets, or missing coordinates. The main blockers for expanding are: CORS (would need a proxy server), non-Socrata platforms (CKAN, ArcGIS — different query formats), and datasets without geocoded coordinates (would need server-side geocoding). A backend proxy would unlock the most cities.

## Deployment
- **Guide:** `DEPLOYMENT-GUIDE.md` covers GitHub Pages setup (Part 1) and Supabase database setup (Part 2)
- **Security migrations (run in order in Supabase SQL Editor):**
  1. `RLS-UPGRADE.sql` — RLS policies, admin token, `is_admin()`, `refresh_pin()`
  2. `INSERT-LOCKDOWN.sql` — removes public INSERT, creates `create_pin()` with rate limiting
  3. `ADD-NOTES-FIELD.sql` — adds `notes` column, updates `create_pin()` with `p_notes` parameter
  4. `UPDATE-DETAILS.sql` — creates `update_pin_report()` for inline detail editing
- **All changes go live** by uploading the updated `index.html` to the repo; GitHub Pages rebuilds in 1-2 minutes
- **Hard refresh** (Ctrl+Shift+R / Cmd+Shift+R) is often needed after uploading to bust browser cache

## Style Notes
- Dark theme throughout (background: #0a0a1a)
- Accent color: amber/orange (#f59e0b)
- Permit pin color: blue (#3b82f6) — square shape to distinguish from round user-report pins
- User report pin colors: green (#22c55e), yellow (#eab308), red (#ef4444)
- System fonts (-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif)
- Modal overlays use backdrop blur and gradient backgrounds
- UI is designed to work on both desktop and mobile

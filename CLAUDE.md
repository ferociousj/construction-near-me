# Construction Near Me

## What This Is
A website that lets people find and report active construction sites on a map. Target audience is parents of small kids who want to watch big trucks, plus community members who help by reporting sightings. Brainchild of Jon Sung (https://linktr.ee/ferociousj).

## Live Site
Hosted on GitHub Pages. Single `index.html` file — the entire app lives in one file.

## Tech Stack
- **Frontend:** Vanilla HTML/CSS/JS (no framework, no build step)
- **Map:** Leaflet + OpenStreetMap tiles (free, no API key)
- **Geocoding:** Nominatim (OpenStreetMap's geocoder, free, no API key)
- **Database:** Supabase (optional — works in offline/localStorage mode without it)
- **Hosting:** GitHub Pages (free)

## Architecture Decisions

### Single-file app
Everything is in one `index.html` for maximum simplicity. The creator is not a developer, so minimizing files, build tools, and complexity is a priority.

### Supabase integration is optional
The app detects whether Supabase credentials are present (hardcoded or in localStorage). If yes, it connects to the shared database. If not, it runs in offline mode where each visitor has their own pins in localStorage. This was a deliberate choice so the site works immediately on GitHub Pages without any backend setup.

### Hardcoded credentials
Supabase URL and anon key can be hardcoded in two clearly labeled constants at the top of the script section:
```javascript
const HARDCODED_URL = '';
const HARDCODED_KEY = '';
```
The boot() function checks these with .trim() and .length > 0 to avoid whitespace/null issues. If filled in, the setup screen never appears.

### Security model (intentionally simple)
- Admin password gates the UI only — it's checked client-side in JavaScript
- In database mode, the password is stored in a `settings` table readable by anyone with the anon key
- Row Level Security on Supabase allows anyone to read, insert, update, and delete pins
- This is accepted as fine for a small community project. Real auth (Supabase Auth) is a future upgrade if needed.

### Admin interface
- Accessed via a barely-visible "admin" text link in the top-right of the header
- Default password: `chocolaterain`
- Features: sortable/filterable table, bulk delete, bulk set freshness, change password
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
```

### Settings table (Supabase)
```sql
key TEXT (primary key)
value TEXT
```
Currently only stores `admin_password`.

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
- Pin detail popups with machinery tags, good view flag, date, "Still there?" button
- "Still there?" resets date_reported to now (resets freshness)
- Search & Filter panel: location search (Nominatim geocoding), freshness range, good view filter, machinery filter
- Admin interface: sortable/filterable table, select all/deselect all, bulk delete, bulk set freshness, change password
- About overlay with attribution to Jon Sung
- Supabase integration with offline fallback
- Setup screen for first-time Supabase credential entry (skipped if hardcoded)
- Sample pins around San Francisco pre-loaded

## Features NOT Yet Implemented
- Directions to a pin from user's location (noted as nice-to-have in design doc)
- Public construction data sources (open question in design doc)

## Future Options (discussed but not built yet)

### Google Maps
Swap Leaflet/OSM for Google Maps JavaScript API. Requires Google Cloud account, API key, and billing info (free tier covers ~28,000 map loads/month). Would also swap Nominatim for Google Geocoding API. Ready to implement when requested.

### Custom domain
Cloudflare Registrar recommended as cheapest option (~$10-11/year for .com, sold at wholesale cost). `constructionnear.me` noted as a clever option. GitHub Pages supports custom domains — just needs a CNAME file and DNS config.

### Stronger security
Upgrade from client-side password check to Supabase Auth with real user accounts. Admin would be a specific authenticated user. Row Level Security would restrict delete/update to that user only.

## Deployment Guide
A step-by-step guide exists as `DEPLOYMENT-GUIDE.md` covering:
- Part 1: GitHub Pages setup (10 min, no technical experience needed)
- Part 2: Supabase database setup (15 min, copy-paste SQL)
- Optional: Hardcoding credentials, troubleshooting

## Style Notes
- Dark theme throughout (background: #0a0a1a)
- Accent color: amber/orange (#f59e0b)
- Pin colors: green (#22c55e), yellow (#eab308), red (#ef4444)
- System fonts (-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif)
- Modal overlays use backdrop blur and gradient backgrounds
- UI is designed to work on both desktop and mobile

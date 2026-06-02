# TMDB Watchlist by HTTP

Monitors TV series and movies from a TMDB watchlist via the TMDB API v3. Discovers entries automatically via LLD and tracks release dates, season count, production status, and physical/digital releases.

## Requirements

- Zabbix 7.0+
- TMDB account with an active API subscription (https://www.themoviedb.org/settings/api)

## Authentication Setup

Three credentials are required: API Key, Session ID, and Account ID.

### 1. Get API Key

Go to TMDB Settings, API, and copy the **API Key (v3 auth)**, a short alphanumeric string.

### 2. Generate Session ID

The session ID authenticates watchlist requests. It does not expire unless manually revoked in TMDB settings.

```bash
# Step 1: Request a request token
curl -s "https://api.themoviedb.org/3/authentication/token/new?api_key=YOUR_API_KEY"
# Note the "request_token" value

# Step 2: Approve the token in the browser
# Open: https://www.themoviedb.org/authenticate/YOUR_REQUEST_TOKEN

# Step 3: Exchange for session ID
curl -s -X POST "https://api.themoviedb.org/3/authentication/session/new?api_key=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"request_token":"YOUR_REQUEST_TOKEN"}'
# Note the "session_id" value
```

### 3. Get Account ID

Requires API Key and Session ID from the steps above.

```bash
curl -s "https://api.themoviedb.org/3/account?api_key=YOUR_API_KEY&session_id=YOUR_SESSION_ID"
```

The response starts with `"id"`:

```json
{
    "id": 12345678,
    "name": "Your Name",
    "username": "youruser",
    ...
}
```

Use the numeric `id` value as `{$TMDB.ACCOUNT_ID}`.

## Zabbix Macros

| Macro | Default | Description |
|-------|---------|-------------|
| `{$TMDB.API_KEY}` | | API Key (v3) |
| `{$TMDB.SESSION_ID}` | | Session ID |
| `{$TMDB.ACCOUNT_ID}` | | Numeric account ID |
| `{$TMDB.SEASON.NEW_TRIGGER}` | `1` | Minimum season count increase to alert |
| `{$TMDB.SERIES.STALE_DAYS}` | `180` | Days since last episode before stale alert (requires in_production=1) |
| `{$TMDB.MOVIE.SOON_DAYS}` | `7` | Days before theatrical release to alert |
| `{$TMDB.PHYSICAL.SOON_DAYS}` | `14` | Days before physical release (DVD/Blu-ray) to alert |
| `{$TMDB.DIGITAL.SOON_DAYS}` | `7` | Days before digital release to alert |
| `{$TMDB.COUNTRY}` | `DE` | ISO 3166-1 country code for physical/digital release lookups |

## API Calls

All requests use the v3 API key as a query parameter. Session ID is only required for watchlist endpoints.

| Endpoint | Interval | Purpose |
|----------|----------|---------|
| `GET /3/account/{ACCOUNT_ID}/watchlist/tv?api_key=...&session_id=...&page=1` | 1h | Fetch TV series watchlist (up to 20 entries) |
| `GET /3/account/{ACCOUNT_ID}/watchlist/movies?api_key=...&session_id=...&page=1` | 1h | Fetch movie watchlist (up to 20 entries) |
| `GET /3/tv/{SERIES_ID}?api_key=...&language=de-DE` | 2h | Series details per discovered series |
| `GET /3/movie/{MOVIE_ID}?api_key=...&language=de-DE&append_to_response=release_dates` | 2h | Movie details including physical/digital release dates per discovered movie |

The watchlist endpoints only return page 1 (up to 20 entries). Entries beyond 20 are not discovered.

## What is Monitored

**Series (per discovered entry):**
- Season count
- Next episode air date
- Production status (in_production flag)
- Series status (Returning Series, Ended, Canceled, etc.)
- Days since last episode (stale detection)

**Movies (per discovered entry):**
- Theatrical release date and days until release
- Movie status (Released, In Production, Planned, etc.)
- Production status (derived from status field)
- Physical release date (DVD/Blu-ray, release type 5) for country `{$TMDB.COUNTRY}`
- Digital release date (streaming/download, release type 4) for country `{$TMDB.COUNTRY}`

## Triggers

| Trigger | Severity | Condition |
|---------|----------|-----------|
| Series: new season announced | Info | season count increased by at least `{$TMDB.SEASON.NEW_TRIGGER}` |
| Series: next episode date changed | Info | next_episode_to_air date set or updated |
| Series: production started | Info | in_production changed to true |
| Series: ended | Info | status changed to Ended |
| Series: cancelled | Warning | status = Canceled |
| Series: stale | Warning | days since last episode > `{$TMDB.SERIES.STALE_DAYS}` and in_production = 1 |
| Series watchlist >20 entries | Warning | page 2+ exists, entries not monitored |
| Movie: release date announced | Info | release_date set or changed |
| Movie: releases soon | Info | release within `{$TMDB.MOVIE.SOON_DAYS}` days |
| Movie: released | Info | status changed to Released |
| Movie: production started | Info | status changed to In Production or Post Production |
| Movie: cancelled | Warning | status = Canceled |
| Movie: physical release announced | Info | physical release date set or changed |
| Movie: physical release soon | Info | physical release within `{$TMDB.PHYSICAL.SOON_DAYS}` days |
| Movie: digital release announced | Info | digital release date set or changed |
| Movie: digital release soon | Info | digital release within `{$TMDB.DIGITAL.SOON_DAYS}` days |
| Movie watchlist >20 entries | Warning | page 2+ exists, entries not monitored |
| No data received | Warning | no data from TMDB API for 2 hours |

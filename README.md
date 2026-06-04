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

**Step 1:** Request a token:

```bash
curl -s "https://api.themoviedb.org/3/authentication/token/new?api_key=YOUR_API_KEY"
```

Expected response:

```json
{"success":true,"expires_at":"2026-01-01 12:00:00 UTC","request_token":"abc123..."}
```

Copy the `request_token` value.

**Step 2:** Open this URL in a browser and log in with your TMDB account to approve the token:

```
https://www.themoviedb.org/authenticate/YOUR_REQUEST_TOKEN
```

Replace `YOUR_REQUEST_TOKEN` with the value from Step 1. TMDB will confirm the approval on screen. Step 3 will fail with `Session denied` if this step is skipped.

**Step 3:** Exchange the approved token for a session ID:

```bash
curl -s -X POST "https://api.themoviedb.org/3/authentication/session/new?api_key=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"request_token":"YOUR_REQUEST_TOKEN"}'
```

Replace `YOUR_REQUEST_TOKEN` with the value from Step 1.

Expected response:

```json
{"success":true,"session_id":"abc123def456..."}
```

Copy the `session_id` value.

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
- Episode count (total across all seasons)
- Next episode air date (season and episode number)
- Last episode air date (season and episode number)
- Production status (in_production flag)
- Series status (Returning Series, Ended, Canceled, etc.)
- Days since last episode (stale detection)
- Vote average (informational)

**Movies (per discovered entry):**
- Theatrical release date and days until release
- Movie status (Released, In Production, Planned, etc.)
- Production status (derived from status field)
- Runtime in minutes
- Physical release date (DVD/Blu-ray, release type 5) for country `{$TMDB.COUNTRY}`
- Digital release date (streaming/download, release type 4) for country `{$TMDB.COUNTRY}`
- Vote average (informational)

## Triggers

| Trigger | Severity | Condition |
|---------|----------|-----------|
| Series: new season announced | Info | season count increased by at least `{$TMDB.SEASON.NEW_TRIGGER}` |
| Series: new episode added | Info | episode count increased (TMDB entry added, may precede air date) |
| Series: new episode aired | Info | last_air_date changed; event name shows S__E__ and date |
| Series: next episode date changed | Info | next_episode_to_air date set or updated |
| Series: production started | Info | in_production changed to true |
| Series: ended | Warning | status changed to Ended |
| Series: cancelled | Warning | status changed to Canceled |
| Series: stale | Warning | days since last episode > `{$TMDB.SERIES.STALE_DAYS}` and in_production = 1 |
| Series watchlist >20 entries | Warning | page 2+ exists, entries not monitored |
| Movie: release date announced | Info | release_date set or changed |
| Movie: releases soon | Warning | release within `{$TMDB.MOVIE.SOON_DAYS}` days |
| Movie: released | Info | status changed to Released |
| Movie: production started | Info | status changed to In Production or Post Production |
| Movie: runtime set | Info | runtime changed from 0 to a valid value |
| Movie: cancelled | Warning | status changed to Canceled |
| Movie: physical release announced | Info | physical release date set or changed |
| Movie: physical release soon | Warning | physical release within `{$TMDB.PHYSICAL.SOON_DAYS}` days |
| Movie: digital release announced | Info | digital release date set or changed |
| Movie: digital release soon | Warning | digital release within `{$TMDB.DIGITAL.SOON_DAYS}` days |
| Movie watchlist >20 entries | Warning | page 2+ exists, entries not monitored |
| No data received | Warning | no data from TMDB API for 2 hours |
<img width="2842" height="1422" alt="image" src="https://github.com/user-attachments/assets/51262deb-b977-4701-9db9-7b4f0c2c8ef9" />

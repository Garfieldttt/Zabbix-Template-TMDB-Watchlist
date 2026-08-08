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
| `{$TMDB.MOVIE.SOON_DAYS}` | `7` | Days before theatrical release to alert |
| `{$TMDB.DIGITAL.SOON_DAYS}` | `7` | Days before digital release to alert |
| `{$TMDB.COUNTRY}` | `DE` | ISO 3166-1 country code for physical/digital release and streaming provider lookups |

## API Calls

All requests use the v3 API key as a query parameter. Session ID is only required for watchlist endpoints.

| Endpoint | Interval | Purpose |
|----------|----------|---------|
| `GET /3/account/{ACCOUNT_ID}/watchlist/tv?api_key=...&session_id=...&page=1` | 30m | Fetch TV series watchlist (up to 20 entries) |
| `GET /3/account/{ACCOUNT_ID}/watchlist/movies?api_key=...&session_id=...&page=1` | 30m | Fetch movie watchlist (up to 20 entries) |
| `GET /3/tv/{SERIES_ID}?api_key=...&language=de-DE&append_to_response=watch/providers` | 30m | Series details and streaming providers per discovered series |
| `GET /3/movie/{MOVIE_ID}?api_key=...&language=de-DE&append_to_response=release_dates,watch/providers` | 30m | Movie details including physical/digital release dates and streaming providers per discovered movie |

The watchlist endpoints only return page 1 (up to 20 entries). Entries beyond 20 are not discovered.

## What is Monitored

**Series (per discovered entry):**
- Season count
- Episode count (total across all seasons)
- Next episode as text (`S03E01 'Homecoming' airs 2026-08-15`, or `Not announced`)
- Next episode air date as a Unix timestamp
- Last episode as text (`S02E05 'The Reckoning' (2026-07-20)`, or `No episode aired yet`)
- Last episode air date as a Unix timestamp
- Days since last episode
- Production status (in_production flag)
- Series status (Returning Series, Ended, Canceled, etc.)
- Flatrate streaming providers for country `{$TMDB.COUNTRY}` (JustWatch data)
- Vote average (informational)

**Movies (per discovered entry):**
- Theatrical release date and days until release
- Movie status (Released, In Production, Planned, etc.)
- Production status (derived from status field)
- Runtime in minutes
- Physical release date (DVD/Blu-ray, release type 5) for country `{$TMDB.COUNTRY}`
- Digital release date (streaming/download, release type 4) for country `{$TMDB.COUNTRY}` and days until digital release
- Flatrate streaming providers for country `{$TMDB.COUNTRY}` (JustWatch data)
- Vote average (informational)

Runtime, production status and vote average are collected as items only. They have no triggers, so they show up in Latest data and on the dashboard without generating alerts.

## Triggers

The trigger set is kept deliberately small so that a single event produces a single alert.

| Trigger | Severity | Condition |
|---------|----------|-----------|
| Series: new season announced | Info | season count increased by at least `{$TMDB.SEASON.NEW_TRIGGER}`; event name shows the next episode |
| Series: new episode added | Info | episode count increased; event name shows the next episode |
| Series: ended | Warning | status changed to Ended |
| Series: cancelled | Warning | status changed to Canceled |
| Movie: release date announced or changed | Info | release_date set or changed |
| Movie: releases soon | Average | release within `{$TMDB.MOVIE.SOON_DAYS}` days |
| Movie: cancelled | Warning | status changed to Canceled |
| Movie: physical release date announced or changed | Info | physical release date set or changed |
| Movie: digital release date announced or changed | Info | digital release date set or changed |
| Movie: digital release soon | Warning | digital release within `{$TMDB.DIGITAL.SOON_DAYS}` days |
| Movie: available on streaming | Info | flatrate provider list for `{$TMDB.COUNTRY}` changed and is not empty |
| Series: available on streaming | Info | flatrate provider list for `{$TMDB.COUNTRY}` changed and is not empty |
| No data received | Warning | no data from TMDB API for 2 hours |

There is deliberately no trigger on the movie `status` field turning to `Released`. On TMDB that field is maintained by hand and contributors flip it to `Released` once the film is finished, typically one to two weeks before the actual release date. It means "production complete", not "watchable". Use "releases soon", "digital release soon" and "available on streaming" instead, all of which are driven by real dates or by JustWatch provider data.

Four triggers show a text value from a second item in their event name (next episode, release date, digital release date). Zabbix expression macros such as `{?last(...)}` only resolve numeric items and render as `*UNKNOWN*` on text items, so these triggers carry a trailing expression term of the form `or length(last(/.../item))<0`. That term is always false and never affects when the trigger fires. It only makes the text item the second item of the expression so that `{ITEM.LASTVALUE2}` resolves.
<img width="2842" height="1422" alt="image" src="https://github.com/user-attachments/assets/51262deb-b977-4701-9db9-7b4f0c2c8ef9" />

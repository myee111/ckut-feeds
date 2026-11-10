# ckut-feeds

RSS feeds for CKUT 90.3 FM show pages, which have none that work.

CKUT's playlist pages (e.g. <https://ckut.ca/playlists/OH>) still advertise two feeds in their `<head>`:

```
https://ckut.ca/playlistfeed/OH.xml      -> 404
https://ckut.ca/archivefeed/mp3/OH.xml   -> 404
```

This repo scrapes the same HTML instead and publishes static XML from GitHub Pages:

| File | What it is | For |
| --- | --- | --- |
| `feed-oh.xml` | Episode list with the full tracklist in each description | any RSS reader |
| `podcast-oh.xml` | The same, plus `<enclosure>` pointing at CKUT's own MP3 | podcast apps, download tools |

Nothing is rehosted: audio enclosures are the station's own `archives.ckut.ca` files, resolved through their
`listen.php` redirect.

## Deploy

1. Create a **public** repo and push this folder to `main`. Pages needs the repo public on a free plan.
2. Settings → Pages → Source: *Deploy from a branch* → Branch: `main`, folder `/ (root)`.
3. Actions tab → *Update feeds* → *Run workflow* once, so the first feeds are built with the real base URL.
4. Subscribe to:
   - `https://<user>.github.io/<repo>/feed-oh.xml`
   - `https://<user>.github.io/<repo>/podcast-oh.xml`

Subscribe to those URLs directly. Pasting the CKUT page URL into a reader makes the reader find CKUT's dead
feed links instead.

## How it works

`.github/workflows/feed.yml` runs `scrape.py` hourly (`17 * * * *`), commits the regenerated XML if it changed,
and Pages serves the result. Scheduled runs can be delayed a few minutes; that does not matter for a weekly show.

`scrape.py`:

- fetches `/playlists/OH`, parses each episode row into date + playlist id + archive id;
- fetches `/playlists/shows/<id>` for new episodes and the three most recent (to pick up late additions), and
  caches the tracklist in `cache/tracklists.json`;
- resolves `listen.php?show=<id>d&archive=<id>` through its redirects to the public MP3, caching URL,
  `Content-Length` and duration in `cache/audio.json`, so each episode costs one extra request, once. A few
  episodes were aired without a published playlist, so their show id comes from the archive-player link instead;
- writes both feeds with stable GUIDs (`ckut:oh:playlist:<id>`, `ckut:oh:archive:<id>`), so filling in a
  half-finished playlist later does not re-notify subscribers;
- fails loudly if nothing parses, rather than committing an empty feed.

Steady state is about two requests a week: the index page on every run, one show page per new episode.
`ckut.ca/robots.txt` disallows only `/wp-admin/`.

## Configuration

Environment variables, all optional:

| Variable | Default | Purpose |
| --- | --- | --- |
| `CKUT_SHOW` | `OH` | Show code from the URL `/playlists/<code>` |
| `CKUT_SHOW_NAME` | `Off The Hook` | Human name used in item titles |
| `CKUT_SHOW_SCHEDULE` | Thursday midnight–2am | Line in the channel description |
| `CKUT_MAX_ITEMS` | `40` | How many episodes to publish |
| `CKUT_REFRESH_RECENT` | `3` | Newest N episodes always re-scraped |
| `CKUT_YEARS` | empty | Extra archive pages, e.g. `2025,2024` (they live at `/playlists/OH2025`) |
| `FEED_BASE_URL` | — | Absolute base for `atom:link rel=self`; the workflow derives it |
| `CKUT_USER_AGENT` | `ckut-feed/1.0 …` | Set this to something with a contact URL if you fork it |

For another show: change `CKUT_SHOW`/`CKUT_SHOW_NAME`, then rename the two output files or add a matrix job.

## Local run

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
FEED_BASE_URL=https://example.github.io/ckut-feeds .venv/bin/python scrape.py
```

Outputs land in the repo root; `cache/` is committed on purpose so runs stay cheap.

## Caveats

- Scraping depends on KenzoDB's markup (`div.playlist-archive li`, `#drop_table`). If CKUT changes templates the
  parse guard fails the job instead of silently emptying the feed.
- Episodes marked *"Playlist still being updated"* are published anyway and their descriptions get refreshed on
  later runs. Episodes with no playlist page at all still appear, linked to the show's archive index.
- A two-hour episode is ~117 MB. Podcast apps stream it from CKUT's server, which is the point, but it is their
  bandwidth — do not mirror the files.
- Content is CKUT's. This is a personal-use convenience: every item links back to the playlist page and the feed
  credits the station. Do not publish the tracklists commercially.

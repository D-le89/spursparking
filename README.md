# Tottenham Hotspur Stadium — Event Day Parking Alerts (MVP)

Notifies drivers without a parking permit when Tottenham Hotspur Stadium
event-day restrictions mean they need to move their car, so they can avoid
a Penalty Charge Notice (PCN) from Haringey/Enfield.

## What it does

- **Register tab** — anyone can sign up with their email, CPZ zone, and how
  much advance notice they want.
- **Upcoming events tab** — public view of scheduled events with estimated
  "move your car by" times.
- **Admin tab** — manually add/remove events (matches, concerts, etc.) and
  see who's subscribed.
- **`notifier.py`** — a script meant to run on a schedule (cron) that sends
  two emails per subscriber per event: an advance notice and a same-day
  reminder, timed relative to when restrictions are expected to start.

## Setup

```bash
cd spursparking
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
streamlit run app.py
```

Opens at `http://localhost:8501`. Use the **Admin** tab to add a few
events, then **Register** to subscribe a test email.

## Sending real emails

By default `notifier.py` runs in **dry run mode** (prints what it would
send). To send real emails:

1. Copy `.env.example` to `.env` and fill in SMTP credentials (Gmail
   works with an [App Password](https://support.google.com/accounts/answer/185833)
   — you can't use your normal Gmail password).
2. Export those variables before running, e.g.:
   ```bash
   export $(cat .env | xargs)
   python3 notifier.py
   ```
3. Set up a cron job to run it regularly, e.g. every 30 minutes:
   ```
   */30 * * * * cd /path/to/spursparking && source venv/bin/activate && export $(cat .env | xargs) && python3 notifier.py
   ```

## Pulling events from tottenhamhotspurstadium.com

The Admin tab has a **"Run scraper now"** button (`scraper.py`) that fetches
the official events pages and extracts candidate events (concerts, NFL
games, etc.). A few things worth knowing:

- The main `/events` calendar grid is rendered by JavaScript, so a plain
  HTTP fetch can't see it. The scraper instead follows the "Concerts &
  Events" nav links to each individual event page (e.g. `/events/1069172/
  nfl-2026`), where a "Key Dates" section with real dates is present in
  the static HTML.
- **Football fixtures are not on this site at all** — they live on
  tottenhamhotspur.com. Keep adding matches manually (or wire up a
  football data API per `prompt.md`) alongside the scraper.
- Start **times** are rarely published on these pages, only dates — so
  every scraped result lands in a review queue, not straight into the
  live event list. You confirm the start time and event type before
  anything reaches subscribers.
- Before running this regularly/automatically (e.g. on a cron schedule),
  check `tottenhamhotspurstadium.com/robots.txt` and the site's Terms &
  Conditions yourself. Running it manually or a few times a day from the
  Admin tab is a reasonable starting point; don't turn it into a
  high-frequency crawler.

## Files

- `app.py` — Streamlit UI (register / upcoming events / admin)
- `db.py` — SQLite schema and data access (events, subscribers, sent-log,
  scraped-event review queue)
- `scraper.py` — pulls candidate events from tottenhamhotspurstadium.com
- `rules.py` — restriction-window logic (when roads close, when it's safe
  to assume they've reopened, PCN amounts)
- `notifier.py` — cron-run script that sends the actual emails
- `.env.example` — SMTP config template

## Important accuracy note

The default timing rules in `rules.py` are **planning estimates** based on
published Haringey Council / Tottenham Hotspur guidance (closures from
roughly 2–3 hours before kick-off, clearing roughly 1 hour after full
time). Exact bay-suspension times are only confirmed by Haringey around
7 days before each event and can vary by event size and zone. This app
should be used as an early warning, not the final word — the copy in the
emails already tells subscribers to double-check nearer the date, and
that framing should stay in place if you extend this.

Currently the £160 PCN figure (£80 if paid within 14 days) is what's
quoted in the emails — verify this hasn't changed before relying on it,
since fine amounts are set by the council and do get revised.

## Next steps

See `prompt.md` for a ready-to-use prompt to extend this into a real
product — automatic fixture pulling, SMS alerts, a proper zone/postcode
lookup, and multi-tenant support for other stadiums.

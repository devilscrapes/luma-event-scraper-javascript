<p align="center"><a href="https://apify.com/DevilScrapes/luma-event-discovery"><img src="assets/banner.svg" alt="Lu.ma Event Discovery Scraper — JavaScript example" width="100%"></a></p>

# Luma Event Scraper for Node.js: lu.ma feeds as JSON for apps and calendars

<p align="center">
<a href="https://apify.com/DevilScrapes/luma-event-discovery"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/luma-event-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/luma-event-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

This is a single-file Node.js starter for the **[Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery)**, written as an ES module around the official `apify-client` package. Give it Luma slugs, feed addresses or individual event links and it returns plain objects: event name and canonical link, start and end timestamps with an IANA timezone, online or in-person flag, city and region, ticket pricing in cents, registration totals and the lead host. Fetching the pages is Apify's job; your Express handler, BullMQ worker or scheduled Lambda just starts a run and reads what comes back. The script also leaves `results.json` and `results.csv` on disk when you run it from the command line.

> **Actor:** [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [luma-event-scraper-python](https://github.com/devilscrapes/luma-event-scraper-python)

## 🔥 What you get

- **Feeds and single events in the same list**: a city slug like `sf`, a category like `ai`, the `discover` page or one event link can sit side by side in `urls`; the Actor tells list pages from event pages on its own.
- **Any spelling of the address works**: `lu.ma/...`, `luma.com/...` and bare slugs all resolve to the canonical `https://luma.com/<path>` form.
- **Money as integers**: `price_cents` and `max_price_cents` stay in minor units next to a `currency` code, so totals never drift through float rounding.
- **Turnout signals on every row**: `guest_count`, `ticket_count`, `is_sold_out` and `requires_approval` show how busy and how gated an event is.
- **Host context without extra calls**: the first host's display name, Luma username, verified flag and a LinkedIn link (only when it is a `/in/` profile), plus `host_count` for co-hosted events.
- **Speaker data only when asked**: `featured_guest_count` is always filled; the `featured_guests` list appears only with `includeFeaturedGuests` switched on.
- **Failures stay local**: each page gets up to five attempts with growing pauses on throttling and server errors, and a URL that still fails is logged and skipped while the others finish.

## 🎯 Use cases

### Power a local events widget

Refresh a city feed every few hours from a scheduled function, cache the objects in Redis or SQLite and render the next seven days on your community site. `cover_url` gives each card an image, and `timezone` lets you print local start times correctly.

### Publish a subscribable calendar

Turn the rows into an `.ics` file, as in the snippet below, and serve it from a static bucket. Anyone can subscribe from Google Calendar or Apple Calendar and see the category's events without visiting Luma each day.

### Ping Slack when a topic gets a new event

Store the `event_api_id` values you have already seen, run the Actor on a category slug from a cron container and send only the new ones to a Slack or Discord webhook. A devrel team learns about relevant gatherings the morning they are announced.

### Enrich a CRM with event activity

When a partner or customer organizes an event, write a timeline note to your CRM through its API with the event title, date and `guest_count`. Matching is up to your own rules; the Actor just keeps the event facts structured.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/luma-event-scraper-javascript.git
cd luma-event-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Lu.ma Event Discovery Scraper](https://apify.com/DevilScrapes/luma-event-discovery), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

Without the CLI wrapper, the whole job is four statements: build a client, `call()` the Actor, and `listItems()` on the dataset the run produced.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/luma-event-discovery').call({"urls": ["sf"], "maxEventsPerUrl": 10, "includeFeaturedGuests": false, "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` forwards `input.json` untouched. The bundled example asks for ten events from the `sf` city feed, which is enough to see the shape of the data for a few cents. Put several slugs or event URLs in the `urls` array to cover more feeds in one run; event links ignore `maxEventsPerUrl` because they always yield one object.

```json
{
  "urls": [
    "sf"
  ],
  "maxEventsPerUrl": 10,
  "includeFeaturedGuests": false,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `urls` | array | yes | `["sf"]` | — |
| `maxEventsPerUrl` | integer | no | `100` | — |
| `includeFeaturedGuests` | boolean | no | `false` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

The objects are flat, except `featured_guests`, which is an array only when you enable it. Timestamps are ISO-8601 strings and money fields are integer cents, so `new Date()` and simple division are all the parsing a frontend needs.

| Field | Type | Always present |
|---|---|---|
| `source_url` | string | yes |
| `event_api_id` | string | yes |
| `name` | string | yes |
| `event_url` | string | yes |
| `start_at` | string | yes |
| `end_at` | string or null | no |
| `timezone` | string or null | no |
| `event_type` | string or null | no |
| `location_type` | string or null | no |
| `city` | string or null | no |
| `region` | string or null | no |
| `address` | string or null | no |
| `is_free` | boolean | no |
| `price_cents` | integer or null | no |
| `max_price_cents` | integer or null | no |
| `currency` | string or null | no |
| `is_sold_out` | boolean | no |
| `requires_approval` | boolean | no |
| `guest_count` | integer | no |
| `ticket_count` | integer | no |
| `host_name` | string or null | no |
| `host_username` | string or null | no |
| `host_linkedin` | string or null | no |
| `host_is_verified` | boolean | no |
| `host_count` | integer | no |
| `featured_guest_count` | integer | no |
| `featured_guests` | array or null | no |
| `cover_url` | string or null | no |
| `scraped_at` | string | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.004 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/luma-event-discovery) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

The snippet below converts the saved results into an iCalendar file with Node built-ins only, skipping duplicate events and anything without a start time. Serve `luma.ics` anywhere and calendar apps can subscribe to it.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const stamp = (iso) => new Date(iso).toISOString().replace(/[-:]/g, '').replace(/\.\d{3}/, '');
const escape = (text) => String(text ?? '').replace(/([,;\\])/g, '\\$1').replace(/\n/g, '\\n');

const seen = new Set();
const lines = ['BEGIN:VCALENDAR', 'VERSION:2.0', 'PRODID:-//luma-example//EN'];
for (const row of rows) {
    if (!row.start_at || seen.has(row.event_api_id)) continue;
    seen.add(row.event_api_id);
    const where = row.location_type === 'online' ? 'Online' : [row.address, row.city, row.region].filter(Boolean).join(', ');
    lines.push(
        'BEGIN:VEVENT',
        `UID:${row.event_api_id}@luma-example`,
        `DTSTAMP:${stamp(row.scraped_at ?? row.start_at)}`,
        `DTSTART:${stamp(row.start_at)}`,
        ...(row.end_at ? [`DTEND:${stamp(row.end_at)}`] : []),
        `SUMMARY:${escape(row.name)}`,
        `LOCATION:${escape(where)}`,
        `URL:${row.event_url}`,
        'END:VEVENT',
    );
}
lines.push('END:VCALENDAR');
await writeFile('luma.ics', lines.join('\r\n') + '\r\n');
console.log(`wrote ${seen.size} events to luma.ics`);
```

## ❓ FAQ

### Can I call this from a serverless function?

Yes, but mind the timeout. `call()` waits for the run, which is fine for a few feeds; for larger inputs start the run with `start()`, return immediately and collect the dataset later from an Apify webhook or a second invocation.

### Does it work with TypeScript?

Yes. The client package ships its own typings, so the run object is typed out of the box. Dataset items are generic records, though, so write a small `LumaEvent` interface for the fields your code reads and cast `items` to it.

### Are lu.ma and luma.com links treated the same?

Yes. Both hosts, with or without `https://`, and bare slugs like `ai` are rewritten to `https://luma.com/...` before fetching, and `event_url` in the output always uses that host.

### What does a run cost if a feed is empty?

Only the start fee. Charging happens per event written, so a slug that resolves to no events, or a URL that fails after its retries, adds nothing beyond the per-run charge.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/luma-event-discovery) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Lu.ma Event Discovery Scraper on Apify Store](https://apify.com/DevilScrapes/luma-event-discovery)
- [Python version of this example](https://github.com/devilscrapes/luma-event-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Meetup Events Scraper](https://apify.com/DevilScrapes/meetup-events-scraper) — the same kind of rows from Meetup keyword searches, with RSVP totals and host groups.
- [Eventbrite Events Scraper](https://apify.com/DevilScrapes/eventbrite-events-scraper) — ticketed Eventbrite listings when your market lives there instead.
- [Meetup Group Scraper](https://apify.com/DevilScrapes/meetup-group-scraper) — a full upcoming and past event history for chosen Meetup groups.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/luma-event-discovery) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Luma (lu.ma).

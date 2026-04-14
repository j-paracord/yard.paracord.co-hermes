---
name: ghost-research
description: Read-only access to ghost.build Postgres — GitHub trending repos, Hacker News data, and other daily research signals for ambient awareness of what's happening in software today.
---

# ghost-research

The `ghost.build` Postgres instance aggregates daily signals from public sources. Use it when you want a grounded view of what's trending before making recommendations, writing posts, or picking libraries.

Connection comes from the `GHOST_DATABASE_URL` env var. You query via the `psql` client or the `query` helper script bundled with this skill.

## When to use

- The user asks "what's hot on HN / GitHub right now?"
- You need to pick between libraries and want a popularity signal
- You're drafting a post/digest and need source material
- Any open-ended "what's going on in X" question where recency matters

Do **not** use this skill for:
- General web search (use a web search tool)
- The user's own project data
- Anything requiring writes — this connection is read-only

## Tables

### `github_trending`
Daily snapshot of GitHub's trending repos. One row per repo per capture.

| column        | type       | notes                                    |
|---------------|------------|------------------------------------------|
| `captured_at` | timestamptz| when the snapshot was taken (UTC)        |
| `repo`        | text       | `owner/name`                             |
| `language`    | text       | primary language, nullable               |
| `stars`       | int        | total stars at capture time              |
| `stars_today` | int        | delta reported by GitHub for that window |
| `description` | text       | repo description                         |
| `url`         | text       | canonical repo URL                       |

### `hn_items`
Firehose of Hacker News posts + comments (partial — stories and top-level comments kept).

| column        | type       | notes                                    |
|---------------|------------|------------------------------------------|
| `id`          | bigint     | HN item id                               |
| `type`        | text       | `story`, `comment`, `job`, `poll`        |
| `by`          | text       | author handle                            |
| `created_at`  | timestamptz| submission time                          |
| `title`       | text       | stories only                             |
| `url`         | text       | external URL, stories only               |
| `score`       | int        | points at last ingest                    |
| `descendants` | int        | comment count, stories only              |
| `text`        | text       | body (comments / Ask HN), HTML-escaped   |
| `parent`      | bigint     | parent item id for comments              |

More tables get added over time. Run `\dt` in psql or query `information_schema.tables` when in doubt.

## How to query

The bundled `query` helper is a thin `psql` wrapper that loads `GHOST_DATABASE_URL`, sets a read-only session, and prints aligned results:

```bash
./skills/ghost-research/query "SELECT repo, stars_today FROM github_trending WHERE captured_at > now() - interval '1 day' ORDER BY stars_today DESC LIMIT 20"
```

For ad-hoc exploration:

```bash
psql "$GHOST_DATABASE_URL" -c "\dt"
psql "$GHOST_DATABASE_URL" -c "\d github_trending"
```

## Useful recipes

**Top trending repos in the last 24h:**
```sql
SELECT repo, language, stars_today, description
FROM github_trending
WHERE captured_at > now() - interval '1 day'
ORDER BY stars_today DESC NULLS LAST
LIMIT 25;
```

**Top HN stories from today:**
```sql
SELECT id, title, score, descendants, url
FROM hn_items
WHERE type = 'story'
  AND created_at > now() - interval '1 day'
ORDER BY score DESC
LIMIT 25;
```

**New languages bubbling up this week:**
```sql
SELECT language, COUNT(*) AS appearances, SUM(stars_today) AS momentum
FROM github_trending
WHERE captured_at > now() - interval '7 days'
  AND language IS NOT NULL
GROUP BY language
ORDER BY momentum DESC
LIMIT 15;
```

## Guardrails

- Connection is read-only; `INSERT`/`UPDATE`/`DELETE` will fail. Don't retry — just re-read what you need.
- Data may be hours stale depending on ingest cadence. Prefer `captured_at` / `created_at` filters to ensure freshness.
- Result sets can be huge — always `LIMIT` in exploratory queries.

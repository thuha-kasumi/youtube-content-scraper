# Multi-Platform Content Scraper (YouTube + TikTok)

A reusable Streamlit-based scraper for collecting YouTube and TikTok video metadata, channel/creator metadata, transcripts, and comments for portfolio analytics projects. The tool is designed for topic-based research: change the search keywords, run a new scrape, and store normalized results in a shared PostgreSQL database such as Neon.

> Current production path: YouTube scraping on Streamlit Cloud + Neon PostgreSQL. TikTok is kept in the app design, but browser-based scraping is best run locally or in a backend environment that supports Firefox/geckodriver.

---

## Features

| Feature | Detail |
|---|---|
| **Reusable topic workflow** | Run different research topics by changing title, channel/author, transcript, and comment keywords. |
| **YouTube API integration** | Uses YouTube Data API v3 for search, video metadata, comments, and channel metadata. |
| **PostgreSQL data layer** | Stores videos, comments, channels, transcripts, projects, and run logs in Neon/PostgreSQL. |
| **Target-based comment collection** | Set a target number of comments; scraper stops when the target is reached or matching videos are exhausted. |
| **Incremental saving** | Saves comment batches during scraping to reduce data-loss risk. |
| **Duplicate prevention** | Uses stable IDs and database constraints to avoid re-saving the same videos/comments. |
| **Keyword filters** | Supports AND-style filters for title/description, channel/author, transcript, and comments. |
| **Date and popularity filters** | Filter by publish date range, minimum views, and minimum video comments. |
| **Quota tracking** | Displays YouTube API quota usage estimates during the session. |
| **Preview tables** | Shows videos, channels, and sample comments before final metadata save. |
| **Portfolio-ready logging** | Captures run metadata such as platform, keywords, user, quota usage, and result counts. |

---

## Data Layer

This project now uses PostgreSQL instead of Google Sheets as the primary storage layer. Neon is recommended because it is easy to share with collaborators and works well with Streamlit Cloud.

Recommended core tables:

| Table | Purpose |
|---|---|
| `projects` | One row per analysis project or topic. |
| `scrape_runs` | One row per scraper execution. |
| `channels` | One row per YouTube channel or TikTok creator. |
| `videos` | One row per video. |
| `transcripts` | One row per video transcript, when available. |
| `comments` | One row per comment. |
| `project_videos` | Many-to-many bridge between projects and videos. |
| `project_comments` | Many-to-many bridge between projects and comments. |

This structure allows one raw video/comment to be reused across multiple projects without duplication.

---

## Repository Structure

```text
content-scraper/
├── app.py                         # Main Streamlit app
├── requirements.txt               # Streamlit Cloud dependencies
├── README.md                      # Project documentation
├── LICENSE                        # MIT license
├── secrets.toml.example           # Example Streamlit secrets file
└── requirements_tiktok_local.txt   # Optional local TikTok dependencies
```

---

## Setup

### 1. Clone the project

```bash
git clone https://github.com/YOUR_USERNAME/portfolio-youtube-tiktok-scraper.git
cd portfolio-youtube-tiktok-scraper
python -m venv .venv
# Activate .venv for your operating system, then:
python -m pip install -r requirements.txt
```

### 2. Create a Neon PostgreSQL database

1. Create a Neon project.
2. Create or use the default database, for example `neondb`.
3. Copy the pooled connection string.
4. Add the connection string to Streamlit secrets as `DATABASE_URL`.

Example:

```toml
YOUTUBE_API_KEY = "YOUR_YOUTUBE_DATA_API_V3_KEY"
DATABASE_URL = "postgresql+psycopg2://USER:PASSWORD@HOST/DATABASE?sslmode=require"
```

Do not commit `.streamlit/secrets.toml` to GitHub.

### 3. Create the database schema

The app creates or patches its tables automatically before scraping, saving, or
exporting. No separate SQL file is required. The database role must be allowed to
create the schema, tables, and `pgcrypto` extension.

### 4. Create a YouTube Data API key

1. Open Google Cloud Console.
2. Enable **YouTube Data API v3**.
3. Create an API key.
4. Add the key to Streamlit secrets as `YOUTUBE_API_KEY`.

---

## Streamlit Cloud Deployment

Use the included `requirements.txt` at the repository root, next to `app.py`.
It includes the Excel export and optional Reddit dependencies as well as the
core YouTube and database packages.

Set the Streamlit entrypoint to `app.py` and add your real credentials in the
app's private Secrets settings. For local use, copy `secrets.toml.example` to
`.streamlit/secrets.toml`, fill in your credentials, and run:

```bash
streamlit run app.py
```

PostgreSQL is required for scraping, project saving, and exports. The existing
SQLite engine fallback is not a complete offline database implementation.

### Fix for `ModuleNotFoundError: No module named 'psycopg'`

The application uses Psycopg 2 (`psycopg2-binary`), including its fast bulk-insert
helper. SQLAlchemy 2.1 changed the default driver for a plain `postgresql://` URL
from Psycopg 2 to Psycopg 3 (`psycopg`). An unbounded dependency upgrade can
therefore cause an app that previously worked to request an uninstalled driver.
An explicit `postgresql+psycopg://` URL also requests Psycopg 3.

The corrected `get_db_engine()` selects `postgresql+psycopg2` explicitly for
PostgreSQL URLs and preserves the password, host, database, and query options.
The example secrets file now uses the same explicit driver. The requirements
also set Psycopg 2.9.11 as the minimum because it added Python 3.14 support.

To apply the fix to an existing Streamlit Cloud app:

1. Replace `app.py` and `requirements.txt` in the GitHub repository and branch
   used by the deployed app, then commit the changes.
2. In Streamlit's private Secrets settings, change only the URL prefix to
   `postgresql+psycopg2://` if it is currently `postgresql://` or
   `postgresql+psycopg://`. Keep all real connection details and query options.
   The corrected app also normalizes these prefixes automatically.
3. Wait for Streamlit to rebuild the dependencies. If the old error remains,
   use **Manage app → overflow menu → Reboot app** and inspect the build logs.
4. Confirm the interface loads, then try a small YouTube run with 50 search
   candidates, a 50-comment limit per video, and a 500-comment target. Save the
   metadata and check an export. A transcript download can be tested separately.

Changing the example secrets file in GitHub does not update the deployed app's
private secrets. Never replace real credentials with the example placeholders.
This fix does not require deleting the app, changing Python versions, or
resetting the Neon database. Dependency ranges are still flexible; freeze a
known-working environment after validating the deployed app for repeatable builds.

Validation performed for this fix: on Python 3.12, the original app reproduced
`No module named 'psycopg'` with SQLAlchemy 2.1.6; the corrected app rendered its
interface without exceptions using Streamlit 1.66.1 and psycopg2-binary 2.9.13.
Four PostgreSQL URL forms were checked for driver loading and preservation of
encoded passwords and TLS query options. These were startup tests with dummy
credentials; live Neon access, YouTube collection, saving, and exports still
require a deployment test with your own credentials. Python 3.14 runtime behavior
was not tested here.

References:

- [SQLAlchemy 2.1 PostgreSQL driver change](https://docs.sqlalchemy.org/en/21/changelog/migration_21.html#default-postgresql-driver-changed-to-psycopg-psycopg-3)
- [Psycopg release notes](https://www.psycopg.org/docs/news.html)
- [Streamlit app updates and dependencies](https://docs.streamlit.io/deploy/streamlit-community-cloud/manage-your-app)

Do not include `streamlit-gsheets-connection` in the deployed requirements. The app now uses PostgreSQL.

---

## Usage

1. Open the Streamlit app.
2. Select the platform.
3. Enter your name for run logging.
4. Add keyword filters.
5. Optionally enable publish-date filtering.
6. Set minimum views or minimum video comments.
7. Set target comments and scan pool size.
8. Run the scrape.
9. Review preview tables.
10. Save final metadata to PostgreSQL.

Comments are saved incrementally during collection. Final save stores videos, channels, transcripts, and the run log.

---

## Suggested Research Workflow

For each portfolio topic:

1. Create a project label, such as `Vietnam QSR Market Entry`.
2. Run several keyword groups, such as:
   - `Vietnam street food`
   - `McDonald's Vietnam`
   - `KFC Vietnam`
   - `foreign fast food Vietnam`
   - `Vietnam food delivery`
3. Collect broad raw data first.
4. Clean, standardize, translate, and classify comments later.
5. Analyze themes such as price sensitivity, taste expectations, localization, brand trust, service quality, and convenience.

---

## YouTube Quota Notes

Approximate quota costs:

| Operation | Approximate cost |
|---|---:|
| Search request | 100 units |
| Video metadata batch | 1 unit per up to 50 videos |
| Comment page | 1 unit per up to 100 comments |
| Channel metadata | 1 unit per channel |

Budget for search requests as well as comment pages, metadata, and channels.
Even one search request uses 100 units in this app's tracker, so a run that searches
for videos and then fetches 1,500 comments should not be budgeted at 50-100 units.

---

## TikTok Notes

TikTok scraping is browser-automation based and may not run reliably on Streamlit Cloud. Keep TikTok integrated in the interface for a unified workflow, but run TikTok collection locally or on a backend that supports browser automation.

Install the optional local requirements (they include the base requirements):

```bash
python -m pip install -r requirements_tiktok_local.txt
```

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `No module named 'psycopg'` | SQLAlchemy selected Psycopg 3, while the app installs Psycopg 2 | Use the corrected `app.py`, the included requirements, and `postgresql+psycopg2://`. |
| Dependency install fails | Old Google Sheets package still in requirements | Remove `streamlit-gsheets-connection`. |
| Database connection fails | Invalid `DATABASE_URL` or expired Neon password | Re-copy the pooled Neon connection string and confirm `sslmode=require`. |
| No videos matched filters | Filters are too narrow | Remove filters one by one and increase scan pool size. |
| Few comments collected | Matching videos have limited comments | Increase scan pool size or broaden keywords. |
| TikTok fails on Streamlit Cloud | Browser automation unavailable | Run TikTok locally or on a backend server. |

---

## Portfolio Case Study Value

This project demonstrates:

| Competency | Evidence |
|---|---|
| API integration | YouTube Data API v3 usage with quota awareness. |
| Data engineering | PostgreSQL schema, deduplication, incremental inserts. |
| Research tooling | Reusable scraper for multiple business topics. |
| Product thinking | Streamlit UI with filters, previews, and run tracking. |
| Analytics readiness | Normalized tables for later cleaning, NLP, and dashboarding. |

---

## License

MIT License. See `LICENSE`.

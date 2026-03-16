# Facebook Group Feed Scraper

A local web scraper for extracting posts from Facebook Groups using intercepted GraphQL API calls. Includes a browser-based dashboard (FastAPI + HTML) to manage scraping sessions, configure groups, and export results.

---

## What It Does

- Launches a Playwright-controlled Chromium browser using saved session cookies to authenticate automatically
- Intercepts Facebook's internal GraphQL feed requests to capture live session tokens (`fb_dtsg`, `lsd`, `doc_id`)
- Replays those GraphQL requests directly via HTTP to paginate through a group's post feed
- Supports scraping multiple groups sequentially with configurable per-group or global post limits
- Outputs a single combined JSON file with all posts across all groups, each tagged with a `group_id`
- Dashboard UI at `localhost:8000` for managing groups, launching scrapes, and exporting results

---

## Tech Stack

### Browser Automation
- **Playwright (Python sync API)** — launches a non-headless Chromium browser, loads saved cookies, navigates to group pages, and intercepts outgoing network requests

### Network / GraphQL
- **Facebook GraphQL endpoint**: `POST https://www.facebook.com/api/graphql/`
- **Query**: `GroupsCometFeedRegularStoriesPaginationQuery` identified via the `fb_api_req_friendly_name` field in the POST body
- **Session tokens captured from intercepted requests**:
  - `fb_dtsg` — CSRF token, session-wide, same across all requests in a session
  - `lsd` — per-page-load token sent as both a form field and `x-fb-lsd` header
  - `doc_id` — stable query hash identifying the specific GraphQL query (changes infrequently)
  - `group_id` — extracted from the GraphQL variables of the intercepted request
- **Response format**: newline-delimited JSON chunks (streaming GraphQL). Facebook returns multiple JSON objects per response — an initial frame with post edges, stream frames with individual nodes, and a deferred `page_info` frame with `end_cursor` and `has_next_page`
- **Pagination**: cursor-based via `end_cursor` from `data.page_info`, passed as the `cursor` variable in subsequent requests
- **httpx** — used for replaying GraphQL requests with the captured tokens and cookies

### Backend
- **FastAPI** — REST API backend serving the dashboard and handling scrape control
- **uvicorn** — ASGI server
- **Threading + queue.Queue** — browser thread owns the Playwright lifetime; scrape jobs are signaled via a queue to avoid cross-thread Playwright errors

### Frontend
- Vanilla HTML/JS dashboard (`static/index.html`)
- Polls `/status` every 2 seconds during active scrapes

---

## File Structure

| File | Description |
|---|---|
| `main.py` | Entry point — starts FastAPI via uvicorn and opens the dashboard in a browser tab |
| `server.py` | FastAPI backend — manages app state, browser thread, scrape loop, group queue, and all API endpoints |
| `fb_auth.py` | Playwright browser management — launches Chromium, loads cookies, navigates to groups, intercepts GraphQL requests to capture live session tokens |
| `fb_client.py` | HTTP client — replays GraphQL requests using captured tokens; handles retry logic for network errors and Facebook application errors |
| `fb_parser.py` | Response parser — splits newline-delimited GraphQL response chunks, extracts post nodes, `end_cursor`, and `has_next_page` |
| `fb_extractor.py` | Post data extractor — parses raw GraphQL post nodes into clean dicts; detects shared posts and extracts the child (original) post's fields |
| `fb_config.py` | Configuration — GraphQL variables base, headless mode toggle, fallback token values |
| `fb_session.py` | Cookie save/load utilities |
| `groups.json` | Persistent group queue — stores group URL, ID, name, visibility, members, and max_posts per group |
| `cookies/` | Saved browser session cookies (`fb_session.json`) |
| `output/` | Scraped post JSON files |
| `static/index.html` | Dashboard UI |

---

## Key Fixes Made During Development

### Cross-thread Playwright error
Playwright's sync API cannot be used across threads. Fixed by keeping the browser alive in a single long-lived thread and using `queue.Queue` to signal scrape jobs from the FastAPI thread.

### Pagination stopping early
The GraphQL variables had a hardcoded `group_id` from a different group. Fixed by extracting `group_id` from the intercepted request's variables and passing it through to every subsequent paginated request.

### `1675012` error mid-pagination (`missing_required_variable_value`)
Facebook returns this error when the pagination cursor or `lsd` token becomes stale mid-scrape (typically after 20-30 pages). Fixed by:
1. Detecting the error in the parsed response body (it comes as a 200 OK with an `errors` array, not an HTTP error)
2. Re-navigating to the group page to capture fresh tokens
3. Retrying the same cursor — if it still fails, resetting the cursor to `None` and restarting pagination from the beginning of that group

### `capture_tokens` timeout on retry
When re-capturing tokens mid-scrape, the group page sometimes didn't fire a GraphQL request within the original 20s timeout. Fixed by increasing timeout to 45s and scrolls from 4 to 8 to reliably trigger the feed request.

### Shared post extraction crash
`metadata` in GraphQL post nodes is a list, not a dict. Caused `AttributeError` when iterating. Fixed by searching the list for the item containing `creation_time`.

### Python 3.9 type hint incompatibility
`dict | None` union syntax requires Python 3.10+. Removed the type hint for compatibility.

---

## How to Run

```bash
# Install dependencies
pip3 install -r requirements.txt
playwright install chromium

# Run
python3 main.py
```

Open `http://localhost:8000` — click **Launch Browser**, log in if needed, add group URLs, set post limits, and click **Start All**.

---

## Output Format

Single combined JSON file saved to `output/posts_all_<date>_<time>.json`:

```json
{
  "abc123": {
    "post_id": "abc123",
    "group_id": "1046291836187649",
    "text": "...",
    "actor_name": "...",
    "actor_id": "...",
    "created_at": 1710000000,
    "permalink": "https://www.facebook.com/...",
    "images": ["https://..."]
  }
}
```

---

## License

For personal and research use only. Do not use in violation of Facebook's Terms of Service.

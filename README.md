# Job Alert Bot

An automated job scraper that monitors career pages of 100 companies across **Bulge-Bracket Banks**, **Global & Domestic Boutique Investment Banks**, **Big 4 Transaction Advisory**, and **Asset Management / PMS** firms, and sends new matching job alerts directly to your Telegram. No server required — runs entirely on GitHub Actions, on whatever schedule you set.

Built for a **finance profile** (MBA Finance, CFA candidate) targeting:
- Investment Banking Analyst / Associate
- Equity Research Analyst
- Investment Analyst
- Fixed Income Analyst
- Research Analyst

---

## How It Works

```
GitHub Actions timer fires (every 55 min / custom schedule)
        ↓
Scrape all 100 company career pages
(Workday API / Generic HTML)
        ↓
Filter jobs by keywords matching target roles
(investment banking, equity research, fixed income, analyst, associate, m&a ...)
        ↓
Generate SHA256 ID for each job → compare with seen_jobs.json
        ↓
Send only NEW jobs to Telegram
        ↓
Commit updated seen_jobs.json back to the repo
```

---

## Project Structure

```
Bot/
├── job_scraper.py             # Core application — all scraping, filtering, alerting
├── companies.json             # 100 companies with career URLs and filter keywords
├── seen_jobs.json             # Auto-managed list of already-sent job IDs
├── .github/
│   └── workflows/
│       └── job_alert.yml      # GitHub Actions — schedules and runs the bot
├── README.md                  # This file
└── job_alert_explained.md     # Deep-dive explanation of the workflow YAML
```

---

## File Descriptions

### `job_scraper.py`

The main script. All scraping logic, Telegram integration, and orchestration.

#### Functions

| Function | Purpose |
|---|---|
| `load_seen_jobs()` | Reads `seen_jobs.json`, returns a set of previously seen job IDs |
| `save_seen_jobs(seen)` | Writes the updated set back to `seen_jobs.json` |
| `load_companies()` | Parses `companies.json`, returns list of company configs |
| `job_id(company, title, url)` | Creates a 16-char SHA256 hash as unique ID for a job posting |
| `scrape_workday(company)` | Scrapes Workday ATS REST API (used by Barclays, Deutsche Bank, Franklin Templeton) |
| `scrape_lever(company)` | Scrapes Lever ATS JSON API (supported, not currently used by any listed company) |
| `scrape_microsoft(company)` | Scrapes Microsoft Jobs REST API (supported, not currently used) |
| `scrape_greenhouse(company)` | Scrapes Greenhouse ATS JSON API (supported, not currently used) |
| `get_next_page_url(soup, url)` | Finds the "Next Page" link in HTML for pagination |
| `extract_jobs_from_page(soup, url, name, keywords)` | Extracts job links from a single HTML page using `<a>` tag heuristics, filtered by finance job-title signals |
| `scrape_generic(company)` | Generic HTML scraper with auto-pagination for standard career pages |
| `scrape_jobs(company)` | Router — picks the right scraper based on the company URL format |
| `send_telegram(message)` | Sends a message via Telegram Bot API |
| `format_messages(new_jobs)` | Formats jobs into Telegram messages (4000-char limit per message) |
| `run()` | Orchestrates: load config → scrape → diff → alert → save state |

#### Scraper Routing Logic (`scrape_jobs`)

| URL pattern | Scraper used |
|---|---|
| Starts with `workday:` | `scrape_workday()` |
| Starts with `lever:` | `scrape_lever()` |
| Contains `jobs.careers.microsoft.com` | `scrape_microsoft()` |
| Contains `boards-api.greenhouse.io` | `scrape_greenhouse()` |
| Everything else | `scrape_generic()` |

#### Required Environment Variables

| Variable | Description |
|---|---|
| `TELEGRAM_TOKEN` | Your Telegram bot token (from @BotFather) |
| `TELEGRAM_CHAT_ID` | The chat/group ID to receive alerts |

---

### `companies.json`

Configures all companies to monitor. Each entry has:
```json
{
  "name": "Company Name",
  "career_url": "https://careers.example.com  OR  workday:tenant:server:site  OR  lever:slug",
  "keywords": ["investment banking", "investment banking analyst", "equity research", "fixed income"]
}
```

#### Full Company List (100 companies)

**Global bulge-bracket banks (India operations)**

| Company | Scraper |
|---|---|
| Goldman Sachs | Generic HTML |
| Morgan Stanley | Generic HTML |
| J.P. Morgan | Generic HTML (Oracle Cloud recruiting) |
| Citi | Generic HTML |
| Bank of America | Generic HTML |
| Barclays | Workday API |
| Deutsche Bank | Workday API |
| UBS | Generic HTML |
| Nomura | Generic HTML |
| HSBC | Generic HTML |
| Standard Chartered | Generic HTML |
| Jefferies | Generic HTML |

**Elite global boutiques (India offices)**

| Company | Scraper |
|---|---|
| Rothschild & Co | Generic HTML |
| Moelis & Company | Generic HTML |
| Lazard | Generic HTML |
| Houlihan Lokey | Generic HTML |
| BDA Partners | Generic HTML |
| DC Advisory | Generic HTML (Workable ATS) |
| Investec | Generic HTML |

**Leading Indian full-service investment banks**

| Company | Scraper |
|---|---|
| Kotak Investment Banking | Generic HTML |
| SBI Capital Markets | Generic HTML |
| JM Financial | Generic HTML |
| ICICI Securities | Generic HTML |
| Axis Capital | Generic HTML |
| IIFL Capital Services | Generic HTML |
| Edelweiss Financial Services | Generic HTML |
| Motilal Oswal Investment Banking | Generic HTML |
| YES Securities | Generic HTML |
| Anand Rathi Advisors | Generic HTML |

**Big 4 / consulting-led transaction advisory**

| Company | Scraper |
|---|---|
| EY Transaction Advisory | Generic HTML |
| KPMG Deal Advisory | Generic HTML |
| PwC Deals | Generic HTML |
| Deloitte M&A Advisory | Generic HTML |
| Grant Thornton Bharat | Generic HTML |

**Leading domestic boutique investment banks**

| Company | Scraper |
|---|---|
| Avendus Capital | Generic HTML |
| Ambit Private Limited | Generic HTML |
| Arpwood Capital | Generic HTML |
| Singhi Advisors | Generic HTML |
| o3 Capital | Generic HTML |
| Spark Capital | Generic HTML |
| DAM Capital Advisors | Generic HTML |
| Equirus Capital | Generic HTML |
| Veda Corporate Advisors | Generic HTML |
| MAPE Advisory Group | Generic HTML |
| Unitus Capital | Generic HTML |
| IndigoEdge | Generic HTML |
| RBSA Advisors | Generic HTML |
| Aeka Advisors | Generic HTML |
| InCred Capital | Generic HTML |
| NovaaOne / The Rainmaker Group | Generic HTML |

**Largest by PMS AUM (bank/institution-backed asset managers)**

| Company | Scraper |
|---|---|
| SBI Funds Management | Generic HTML |
| UTI Asset Management | Generic HTML |
| Darashaw & Company | Generic HTML |
| Nippon Life India Asset Management | Generic HTML |
| Enam Asset Management | Generic HTML |
| 360 ONE Portfolio Managers | Generic HTML |
| ASK Investment Managers | Generic HTML |
| 360 ONE Asset Management | Generic HTML |
| Franklin Templeton Asset Management (India) | Workday API |
| Quantum Advisors | Generic HTML |
| ICICI Prudential Asset Management | Generic HTML |
| Kotak Mahindra AMC | Generic HTML |
| Motilal Oswal AMC | Generic HTML |
| Axis Asset Management | Generic HTML |
| HDFC Asset Management | Generic HTML |
| Aditya Birla Sun Life AMC | Generic HTML |
| Tata Asset Management | Generic HTML |
| Nuvama Asset Management | Generic HTML |
| PGIM India Asset Management | Generic HTML |

**Leading independent boutique PMS / Cat III AIF houses**

| Company | Scraper |
|---|---|
| Marcellus Investment Managers | Generic HTML |
| WhiteOak Capital Asset Management | Generic HTML |
| Alchemy Capital Management | Generic HTML |
| Abakkus Asset Manager | Generic HTML |
| ValueQuest Investment Advisors | Generic HTML |
| Unifi Capital | Generic HTML |
| Aequitas Investment Consultancy | Generic HTML |
| Old Bridge Capital Management | Generic HTML |
| Sameeksha Capital | Generic HTML |
| Carnelian Asset Management & Advisors | Generic HTML |
| Buoyant Capital | Generic HTML |
| Renaissance Investment Managers | Generic HTML |
| Girik Capital | Generic HTML |
| Multi-Act Equity Consultancy | Generic HTML |
| Right Horizons Portfolio Management | Generic HTML |
| Purnartha Investment Advisers | Generic HTML |
| SageOne Investment Managers | Generic HTML |
| Basant Maheshwari Wealth Advisers | Generic HTML |
| Stallion Asset | Generic HTML |
| ithought Financial Consulting | Generic HTML |
| Negen Capital Services | Generic HTML |
| True Beacon | Generic HTML |
| Tamohara Investment Managers | Generic HTML |
| Ambit Investment Advisors | Generic HTML |
| Karma Capital Advisors | Generic HTML |
| Waterfield Advisors | Generic HTML |
| Sanctum Wealth | Generic HTML |
| Emkay Investment Managers | Generic HTML |
| Bay Capital Investment Advisors | Generic HTML |
| Helios Capital Asset Management | Generic HTML |
| Centrum Investment Advisers | Generic HTML |

> **Note:** Many boutique PMS / wealth-advisory firms (the last category above) don't run a dedicated ATS or job board — for those, `career_url` points at their official homepage or careers/contact page as a best-effort placeholder. The scraper logs 0 jobs harmlessly for firms with no scrapable listing; it just won't surface postings for them until/unless they publish a proper jobs page.

---

### `seen_jobs.json`

Auto-managed array of 16-character job ID hashes. The bot reads this on startup and writes to it after every run. GitHub Actions commits it back to the repo after each run so state persists.

```json
["aa7334f970c917d2", "b1c2d3e4f5a6b7c8", ...]
```

**To reset** (re-receive all current jobs): Replace file contents with `[]` and commit.

---

### `.github/workflows/job_alert.yml`

Automates the bot via GitHub Actions. See `job_alert_explained.md` for a full line-by-line breakdown.

**Steps:**
1. Checkout repository
2. Setup Python 3.11
3. Install dependencies (`requests`, `beautifulsoup4`, `lxml`)
4. Run `job_scraper.py` with Telegram credentials from GitHub Secrets
5. Commit & push updated `seen_jobs.json`

---

## Setup Guide

### Prerequisites
- GitHub account with Actions enabled
- Telegram account

### Step 1 — Create a Telegram Bot

1. Open Telegram → message [@BotFather](https://t.me/BotFather)
2. Send `/newbot`, follow prompts → copy the **API token** (your `TELEGRAM_TOKEN`)
3. Get your `TELEGRAM_CHAT_ID`:
   - Start a conversation with your bot (send it any message)
   - Visit: `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`
   - Find `"chat":{"id": ...}` — that number is your `TELEGRAM_CHAT_ID`

### Step 2 — Fork / Clone this Repository

```bash
git clone https://github.com/your-username/JOB_ALERT_BOT.git
cd JOB_ALERT_BOT
```

### Step 3 — Add GitHub Secrets

Go to: **Repository → Settings → Secrets and variables → Actions → New repository secret**

| Secret Name | Value |
|---|---|
| `TELEGRAM_TOKEN` | Your Telegram bot token |
| `TELEGRAM_CHAT_ID` | Your chat or group ID |

### Step 4 — Enable GitHub Actions

Go to the **Actions** tab → click **"I understand my workflows, go ahead and enable them"** if prompted.

The bot will now run automatically. To trigger it immediately:
**Actions → Job Alert Bot → Run workflow → Run workflow**

---

## Running Locally

```bash
# Install dependencies
pip install requests beautifulsoup4 lxml

# Set credentials (Windows PowerShell)
$env:TELEGRAM_TOKEN = "your_bot_token"
$env:TELEGRAM_CHAT_ID = "your_chat_id"

# Set credentials (macOS/Linux)
export TELEGRAM_TOKEN="your_bot_token"
export TELEGRAM_CHAT_ID="your_chat_id"

# Run
python job_scraper.py
```

---

## Changing the Schedule

Edit the cron line in `.github/workflows/job_alert.yml`:

```yaml
- cron: '*/55 * * * *'   # current: every 55 minutes
```

**Common examples:**

| Goal | Cron value |
|---|---|
| Every 55 minutes (current) | `*/55 * * * *` |
| Every 30 minutes | `*/30 * * * *` |
| 4× daily (6am, 12pm, 6pm, midnight UTC) | `0 6,12,18,0 * * *` |
| Once daily at 9am UTC | `0 9 * * *` |
| 9am IST daily (= 3:30am UTC) | `30 3 * * *` |

All times are UTC. Use [crontab.guru](https://crontab.guru) to verify.

> **Note on GitHub Actions free tier:** With 100 companies scraped per run (mostly generic HTML pages with a 1-second delay per page), each run takes noticeably longer than a small company list did. Free accounts get 2,000 min/month — favor 30-60 minute intervals (or a few fixed times a day) over anything more frequent to stay within limits.

---

## Adding a New Company

### Standard HTML career page
```json
{
  "name": "Acme Capital",
  "career_url": "https://acmecapital.com/careers",
  "keywords": ["investment banking", "investment banking analyst", "equity research"]
}
```

### Greenhouse ATS (API-based, reliable)
```json
{
  "name": "Acme Capital",
  "career_url": "https://boards-api.greenhouse.io/v1/boards/acme/jobs?content=true",
  "keywords": ["investment banking analyst", "research analyst"]
}
```
Avoid bare `analyst`/`associate` as standalone keywords for large, diversified employers — they'll match every ops/tech/compliance "Analyst" title too. Prefer specific phrases (see Keyword Strategy below).
Find the slug from the company's Greenhouse job board URL: `https://boards.greenhouse.io/{slug}`

### Lever ATS (API-based, reliable)
```json
{
  "name": "Acme Capital",
  "career_url": "lever:acme",
  "keywords": ["investment banking analyst", "research analyst"]
}
```
Find the slug from the company's Lever job board URL: `https://jobs.lever.co/{slug}`

### Workday ATS (API-based, reliable)
```json
{
  "name": "Acme Capital",
  "career_url": "workday:acme:wd5:Careers",
  "keywords": ["investment banking analyst", "research analyst"]
}
```
Find `tenant` and `site` from the company's Workday URL: `https://{tenant}.{wd_server}.myworkdayjobs.com/{tenant}/{site}/jobs`. Large global banks and asset managers often run Workday — check for a `myworkdayjobs.com` redirect when you land on their careers page.

---

## Keyword Strategy

Keywords are matched against **job titles** (case-insensitive substring match). Three keyword sets are used depending on company type:

| Category | Keywords used |
|---|---|
| Banks, global/domestic boutiques, Indian IBs | `investment banking`, `investment banking analyst`, `investment banking associate`, `m&a`, `mergers and acquisitions`, `corporate finance`, `equity capital markets`, `debt capital markets`, `leveraged finance`, `research analyst` |
| Big 4 transaction advisory | `transaction advisory`, `deal advisory`, `m&a`, `mergers and acquisitions`, `due diligence`, `valuation`, `investment banking`, `transaction services` |
| Asset managers / PMS / AIF / wealth advisory | `equity research`, `research analyst`, `investment analyst`, `fixed income`, `credit analyst`, `portfolio analyst`, `fund analyst`, `research associate` |

Titles like `"Investment Banking Analyst"`, `"Equity Research Associate"`, `"Fixed Income Analyst"`, `"M&A Analyst"`, `"Transaction Advisory Associate"` will all be caught. Bare, single-word terms like `analyst`/`associate` were deliberately dropped from the keyword lists — large diversified employers (bulge-bracket banks, Big 4 firms) post huge numbers of unrelated ops/tech/compliance "Analyst" roles, and matching on those tokens alone floods alerts with noise (confirmed by testing against Barclays' live Workday feed: ~150 matches on bare `analyst`/`associate` vs. 9 relevant ones with the phrase-based list above). The generic HTML scraper additionally requires the link text to contain a finance job-title signal (`analyst`, `associate`, `banking`, `equity`, `research`, `credit`, `fixed income`, `portfolio`, `investment`, `finance`, `vice president`, etc. — see `extract_jobs_from_page` in `job_scraper.py`) before a keyword match counts, which gives that scraper double-filtering; the 3 Workday-scraped companies (Barclays, Deutsche Bank, Franklin Templeton) rely on the keyword list alone, which is why precision there matters most.

---

## Telegram Alert Format

```
🚀 3 New Job(s) Found
_May 18, 11:30_

🏢 *Morgan Stanley*
💼 Investment Banking Analyst
🔗 https://www.morganstanley.com/careers/...
📝 _Mumbai, India | Investment Banking_
────────────────────────────

🏢 *Motilal Oswal AMC*
💼 Equity Research Analyst
🔗 https://www.motilaloswal.com/careers/...
📝 _Mumbai, India | Research_
────────────────────────────

🏢 *KPMG Deal Advisory*
💼 Deal Advisory Analyst
🔗 https://kpmg.com/in/en/careers/...
📝 _Gurugram, India | Advisory_
────────────────────────────
```

---

## Design Notes

| Decision | Reason |
|---|---|
| Multiple scraper types | Different companies use different ATS platforms (Workday, Greenhouse, Lever, HTML) |
| SHA256 deduplication | Same job is never alerted twice, even across separate runs |
| Git-based state | No database or external storage needed — `seen_jobs.json` lives in the repo |
| 1-second delays | Rate limiting to avoid IP blocks on career pages |
| Chunked Telegram messages | Splits at 4000 chars to stay within Telegram API limits |
| Keyword filtering | Reduces noise — only titles matching target finance roles are included |
| Homepage placeholders for boutique PMS firms | Many small AIF/PMS shops have no dedicated jobs page; pointing at their homepage keeps the entry future-proof if they add one, at the cost of 0 results today |

---

## Troubleshooting

**No alerts received:**
- Check GitHub Actions logs (Actions tab → select a run → view step logs)
- Confirm `TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` secrets are set correctly
- Make sure you sent a message to the bot first (otherwise Telegram won't allow the bot to message you)

**A company returns 0 jobs:**
- Some sites use JavaScript-rendered search (e.g. UBS's Taleo-based board, J.P. Morgan's Oracle Cloud recruiting portal, many boutique PMS homepages) — BeautifulSoup cannot parse client-side XHR-loaded listings. These will log 0 results but won't break the bot.
- For JS-heavy sites, check whether the company actually runs Greenhouse/Lever/Workday under the hood and switch the URL to the API format if so.

**Greenhouse/Lever slug not working:**
- Visit `https://boards-api.greenhouse.io/v1/boards/{slug}/jobs` in your browser — if it returns JSON, the slug is correct. If 404, try variations (e.g., `companyname`, `company-name`).
- For Lever: visit `https://api.lever.co/v0/postings/{slug}?mode=json`

**Workday tenant/site not working:**
- Test directly: `curl -X POST https://{tenant}.{wd_server}.myworkdayjobs.com/wday/cxs/{tenant}/{site}/jobs -H "Content-Type: application/json" -d '{"appliedFacets":{},"limit":1,"offset":0,"searchText":""}'` — a JSON response with `jobPostings` confirms it's correct. A 404/422 means the tenant or site name is wrong (double-check by watching the URL when you browse the company's public careers page — it often redirects to the `myworkdayjobs.com` domain).
- Beware of same-name collisions: some tenant names on Workday belong to a *different* company than the one you're looking for (e.g. `standard` on Workday is an insurance company, not Standard Chartered Bank).

**Too many alerts at once:**
- This happens on the first run after adding new companies. The bot will send all currently listed jobs once, then only new ones going forward.
- To avoid this, clear `seen_jobs.json` to `[]` only when you want to reset.

**GitHub Actions free minutes exhausted:**
- Reduce frequency further, e.g. from `*/55 * * * *` to a few fixed times a day (`0 6,12,18,0 * * *`).

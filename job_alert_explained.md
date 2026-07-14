# job_alert.yml — Full Explanation

This file is a **GitHub Actions workflow**. It tells GitHub to automatically run the job scraper bot on a schedule — no server, no manual effort needed. It now scrapes 100 finance-sector career pages (banks, boutique investment banks, Big 4 transaction advisory, and asset managers/PMS firms) for Abhinav Kulkarni's target roles: Investment Banking Analyst/Associate, Equity Research Analyst, Investment Analyst, Fixed Income Analyst, and Research Analyst.

---

## Full File

```yaml
name: Job Alert Bot

on:
  schedule:
    - cron: '*/55 * * * *'  # every 55 minutes
  workflow_dispatch:       # allows manual trigger

jobs:
  scrape:
    runs-on: ubuntu-latest

    permissions:
      contents: write

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install requests beautifulsoup4 lxml

      - name: Run scraper
        env:
          TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
        run: python job_scraper.py

      - name: Save seen jobs back to repo
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add seen_jobs.json
          git diff --cached --quiet || git commit -m "Update seen jobs"
          git push
```

---

## Line-by-Line Breakdown

### `name: Job Alert Bot`
The display name of this workflow. You will see this name in the GitHub Actions tab on your repository.

---

### `on:` — What triggers this workflow

```yaml
on:
  schedule:
    - cron: '*/55 * * * *'
  workflow_dispatch:
```

This section defines **when** the workflow runs. There are two triggers:

#### 1. `schedule` — Automatic timer
```yaml
cron: '*/55 * * * *'
```
Runs the bot automatically every 55 minutes, all times in **UTC**.

**Cron format:** `minute  hour  day  month  weekday`

| Field | Value | Meaning |
|---|---|---|
| minute | `*/55` | Every 55 minutes past the hour |
| hour | `*` | Every hour |
| day | `*` | Every day of the month |
| month | `*` | Every month |
| weekday | `*` | Every day of the week |

This spacing matters more now than it did with a small company list: with 100 finance career pages to scrape (most via the generic HTML scraper with a 1-second delay per page/pagination step), a single run takes noticeably longer than scraping a handful of ATS APIs. `*/55` keeps runs from overlapping and stays well within the GitHub Actions free-tier minute budget.

#### 2. `workflow_dispatch` — Manual trigger
Allows you to run the workflow manually anytime from the GitHub Actions tab by clicking **"Run workflow"**. No parameters needed. Useful right after editing `companies.json` to see results immediately instead of waiting for the next scheduled tick.

---

### `jobs:` — The work to do

```yaml
jobs:
  scrape:
    runs-on: ubuntu-latest
```

- `jobs:` — contains all the jobs (units of work) in this workflow
- `scrape:` — the name of this job (just a label, can be anything)
- `runs-on: ubuntu-latest` — tells GitHub to spin up a fresh **Ubuntu Linux virtual machine** to run this job. It is temporary — it gets destroyed after the job finishes.

---

### `permissions:`

```yaml
permissions:
  contents: write
```

By default, GitHub Actions has read-only access to your repo. This line gives the workflow **write permission** so it can commit and push the updated `seen_jobs.json` back to the repository.

Without this, the final `git push` step would fail with a permission error.

---

### `steps:` — The individual steps

Each step runs one after another, in order. If a step fails, the remaining steps are skipped.

---

#### Step 1 — Checkout repo

```yaml
- name: Checkout repo
  uses: actions/checkout@v4
```

Downloads your repository code into the virtual machine so the rest of the steps can access your files (`job_scraper.py`, `companies.json`, `seen_jobs.json`, etc.).

- `uses:` means it runs a pre-built GitHub Action (like a reusable script)
- `actions/checkout@v4` is the official GitHub action for checking out code
- `@v4` is the version of that action

Without this step, the VM would have no files to work with.

---

#### Step 2 — Set up Python

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.11'
```

Installs Python 3.11 on the virtual machine.

- `uses: actions/setup-python@v5` — official GitHub action for installing Python
- `with:` — passes configuration to the action
- `python-version: '3.11'` — specifies exactly which Python version to install

This ensures the bot always runs on Python 3.11 regardless of what the Ubuntu image comes with by default.

---

#### Step 3 — Install dependencies

```yaml
- name: Install dependencies
  run: pip install requests beautifulsoup4 lxml
```

- `run:` — executes a shell command directly on the VM
- Installs the three Python libraries that `job_scraper.py` needs:
  - `requests` — makes HTTP requests to career pages and the Workday API
  - `beautifulsoup4` — parses HTML pages to extract job links (used for ~97 of the 100 companies)
  - `lxml` — fast HTML/XML parser used by BeautifulSoup

---

#### Step 4 — Run scraper

```yaml
- name: Run scraper
  env:
    TELEGRAM_TOKEN: ${{ secrets.TELEGRAM_TOKEN }}
    TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
  run: python job_scraper.py
```

This is the main step — it actually runs the bot.

- `env:` — sets environment variables that the script can read
- `${{ secrets.TELEGRAM_TOKEN }}` — pulls the secret you stored in GitHub Secrets and injects it as an environment variable
- `${{ secrets.TELEGRAM_CHAT_ID }}` — same for the chat ID
- `run: python job_scraper.py` — executes the scraper, which loads `companies.json`, scrapes all 100 entries (routing each to `scrape_workday()` or `scrape_generic()` based on its `career_url` format), filters titles against finance keywords (`investment banking`, `equity research`, `analyst`, `associate`, `fixed income`, etc.), and sends any newly-seen postings to Telegram

The script reads `TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` via `os.environ` internally. Secrets are never visible in logs — GitHub automatically masks them.

---

#### Step 5 — Save seen jobs back to repo

```yaml
- name: Save seen jobs back to repo
  run: |
    git config user.name "github-actions"
    git config user.email "actions@github.com"
    git add seen_jobs.json
    git diff --cached --quiet || git commit -m "Update seen jobs"
    git push
```

After the scraper runs, `seen_jobs.json` has been updated in memory on the VM. This step saves it permanently back to the repository.

Line by line:

| Command | What it does |
|---|---|
| `git config user.name "github-actions"` | Sets the Git author name for the commit |
| `git config user.email "actions@github.com"` | Sets the Git author email for the commit |
| `git add seen_jobs.json` | Stages the updated file |
| `git diff --cached --quiet \|\| git commit -m "Update seen jobs"` | Only commits if the file actually changed (avoids empty commits) |
| `git push` | Pushes the commit to the repository |

The `||` in `git diff --cached --quiet || git commit` means: "if the diff check says nothing changed (quiet/success), skip the commit; if something changed (non-zero exit), run the commit."

---

## Full Execution Flow

```
GitHub timer fires every 55 min (or manual trigger)
        ↓
Fresh Ubuntu VM is created
        ↓
Step 1: Your repo files are downloaded onto the VM
        ↓
Step 2: Python 3.11 is installed
        ↓
Step 3: requests, beautifulsoup4, lxml are installed
        ↓
Step 4: job_scraper.py runs
        - Scrapes all 100 finance company career pages
        - Filters titles by IB/equity research/fixed income/analyst keywords
        - Sends new job alerts to Telegram
        - Updates seen_jobs.json locally on the VM
        ↓
Step 5: Updated seen_jobs.json is committed and pushed to GitHub
        ↓
VM is destroyed
```

---

## How to Change the Schedule

Edit line 5 in the file. The format is:
```
cron: 'minute hour day month weekday'
```

**Common examples:**

| Goal | Cron value |
|---|---|
| Once daily at 9am UTC | `0 9 * * *` |
| Every 2 hours | `0 */2 * * *` |
| 9am and 5pm UTC (weekdays only) | `0 9,17 * * 1-5` |
| 9am IST daily (= 3:30am UTC) | `30 3 * * *` |
| Every 55 minutes (current) | `*/55 * * * *` |

All times must be in **UTC**. For IST, subtract 5 hours 30 minutes.
Use [crontab.guru](https://crontab.guru) to verify your expression.

With 100 companies being scraped per run, prefer a schedule of every 30-60 minutes (or a few fixed times a day) over anything more frequent, to stay within the free-tier 2,000 minutes/month.

---

## Where to Set Secrets

Go to your GitHub repository:
**Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|---|---|
| `TELEGRAM_TOKEN` | Your bot token from @BotFather |
| `TELEGRAM_CHAT_ID` | Your Telegram chat or group ID |

These are encrypted and only exposed to the workflow during a run. They are never shown in logs.

---

## How to Trigger Manually

1. Go to your repository on GitHub
2. Click the **Actions** tab
3. Click **Job Alert Bot** in the left sidebar
4. Click **Run workflow** (top right)
5. Click the green **Run workflow** button

This is useful for testing or getting alerts immediately without waiting for the schedule — especially right after editing `companies.json`.

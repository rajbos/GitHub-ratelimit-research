# GitHub Rate Limit Monitor

A tool that samples your GitHub API rate limit every 15 minutes, stores the data long-term in a dedicated `data` branch, and displays it on an interactive GitHub Pages website so you can identify which workflows are consuming your API quota.

## Features

- **Automated sampling** — a GitHub Actions workflow runs every 15 minutes and records your rate limit (core, search, GraphQL) using your Personal Access Token.
- **Long-term storage** — data is kept in an orphan `data` branch as a growing CSV file (`data/ratelimit.csv`), separate from your source code.
- **Interactive dashboard** — a GitHub Pages site (`docs/`) shows:
  - A time-series chart of API calls remaining over time.
  - A bar chart of API calls consumed per 15-minute window (highlighted in red/yellow when high).
  - Time range selector (24 h, 3 d, 7 d, 30 d, all time).
  - Click any data point to see which workflow runs were active during that window.

## Setup

### 1. Create a Personal Access Token

Create a [fine-grained PAT](https://github.com/settings/tokens) (or classic PAT) with at minimum:
- `repo` scope (to read workflow runs)
- `read:user` scope (to read user repos)

### 2. Add the PAT as a repository secret

In your repository settings → *Secrets and variables* → *Actions*, create a secret named **`PAT`** containing your token.

### 3. Enable GitHub Pages

In your repository settings → *Pages*, set the source to **Deploy from a branch**, branch **`main`**, folder **`/docs`**.

### 4. Trigger the first run

Go to *Actions* → *Check GitHub Rate Limit* → *Run workflow* to collect the first data point and create the `data` branch automatically.

### 5. View the dashboard

Open `https://<your-username>.github.io/GitHub-ratelimit-research/` to see your rate limit history.

## How it works

```
Every 15 minutes:
  [Workflow] → GitHub API /rate_limit  →  data branch: data/ratelimit.csv
             → GitHub API /repos       →  data branch: data/runs/<timestamp>.json
                         (recent workflow runs)
```

The website fetches the CSV from the `data` branch raw URL and renders it with [Chart.js](https://www.chartjs.org/).
When you click a data point the site loads the pre-collected workflow-runs snapshot for that checkpoint,
or (if you provide your PAT in the dashboard) queries the GitHub API live.

# GitHub Rate Limit Monitor

A tool that samples your GitHub API rate limit every 15 minutes, stores the data long-term in a dedicated `data` branch, and displays it on an interactive GitHub Pages website so you can identify which workflows are consuming your API quota.

## Features

- **Automated sampling** — a GitHub Actions workflow runs every 15 minutes and records your rate limit (core, search, GraphQL) using your Personal Access Token.
- **Long-term storage** — data is kept in an orphan `data` branch as a growing CSV file (`data/ratelimit.csv`), separate from your source code.
- **Interactive dashboard** — a GitHub Pages site (`docs/`) shows:
  - A time-series chart of API calls remaining over time.
  - A bar chart of API calls consumed per 15-minute window (highlighted in red/yellow when high).
  - Time range selector (24 h, 3 d, 7 d, 30 d, all time).
  - Time window selection on both charts: drag across the bars, click a bar for its whole hour, double-click for just that bar, use the presets, or enter exact times. The window defaults to the latest hour and the charts can zoom to it.
  - Analysis of the selected window: workflows ranked by how their activity lines up with API usage, a filterable list of runs, and bars that light up for the workflow or filter you pick.

## Setup

### 1. Create a Personal Access Token

Create a [fine-grained PAT](https://github.com/settings/tokens) (or classic PAT) with at minimum:
- `repo` scope (to read workflow runs)
- `read:user` scope (to read user repos)

### 2. Add the PAT as a repository secret

In your repository settings → *Secrets and variables* → *Actions*, create a secret named **`PAT`** containing your token.

Also create a secret named **`SNAPSHOT_LABEL_KEY`** containing a random value, for example the output of `openssl rand -hex 32`. Snapshots on the `data` branch are public, so private and internal repos appear as `private-<12 hex characters>` and their workflows as `workflow-<12 hex characters>`, with no real workflow names or IDs, run links or human actors. Without this secret, runs of those repos are left out of the snapshots.

To check whether a label stands for a repo, or for a workflow using `owner/repo#<workflow id>`:

```bash
printf '%s' 'owner/repo' | openssl dgst -sha256 -hmac "$SNAPSHOT_LABEL_KEY" -r | cut -c1-12
```

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
When you select a time window the site loads the pre-collected workflow-runs snapshots for that window and removes duplicate runs.
If the snapshots leave gaps and you saved your PAT in the dashboard, you can fill those gaps from the GitHub API.

### Reading the workflow estimates

The rate limit API only reports totals, so the dashboard estimates. It splits the calls in each bar across the runs active in it, weighted by run time, and compares a workflow's average calls per bar when active and when idle.
Treat the result as a list of suspects to check, not as a measurement:

- Runs that use `GITHUB_TOKEN` do not count against this quota, while calls made outside Actions with the same account do.
- Only the 20 most recently pushed repos are sampled, with at most 100 runs per repo per snapshot (10 in older snapshots).
- Each snapshot looks back to the previous sample, by up to 6 hours, and records any failed lookups so the next one can cover that window. Older snapshots only look back 20 minutes, so runs that started and finished in a longer gap are missing. The dashboard warns you when this affects the selected window.
- When a quota reset falls inside a bar, calls made just before the reset are not known, so that bar shows a lower bound.

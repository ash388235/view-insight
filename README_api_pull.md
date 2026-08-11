# Feeding the dashboards with real data

The dashboards are self-contained HTML files with the CSV data embedded in
`<script type="text/csv">` blocks. To refresh with real data: pull the data with the
API calls below, write the same CSV files into `data/`, then re-run `build_site.py`
(it re-injects the CSVs into the HTML). Nothing else changes — the CSV **column names
are the contract**.

All examples use environment variables for tokens — never hardcode them:

```bash
export GITHUB_TOKEN=ghp_xxx          # GitHub PAT: repo scope (or fine-grained: Contents+Metadata read)
export JIRA_EMAIL=you@bank.com
export JIRA_TOKEN=xxxx               # Jira Cloud API token (id.atlassian.com > Security > API tokens)
export JIRA_BASE=https://acmebank.atlassian.net
```

---

## 1. GitHub — commits per developer (last 3 months)

Target file: `data/commits.csv`
Columns: `sha, repo, author_login, author_name, commit_date, message, additions, deletions, files_changed`

### 1a. List commits per repo (1 call per repo per page)

```
GET https://api.github.com/repos/{owner}/{repo}/commits?since=2026-05-13T00:00:00Z&per_page=100&page=1
Headers:
  Authorization: Bearer $GITHUB_TOKEN
  Accept: application/vnd.github+json
  X-GitHub-Api-Version: 2022-11-28
```

Field mapping from each element of the response array:

| CSV column | JSON path |
|---|---|
| sha | `.sha` |
| repo | (the repo you queried) |
| author_login | `.author.login` (fall back to `.commit.author.email` if `.author` is null) |
| author_name | `.commit.author.name` |
| commit_date | `.commit.author.date` |
| message | `.commit.message` (first line) |

Pagination: keep incrementing `page` until you get fewer than 100 results (or follow the
`Link: rel="next"` response header). 90 days × 6 repos is typically a handful of pages each.

### 1b. Additions / deletions / files per commit (1 call per commit)

The list endpoint does NOT include stats. For each sha:

```
GET https://api.github.com/repos/{owner}/{repo}/commits/{sha}
```

| CSV column | JSON path |
|---|---|
| additions | `.stats.additions` |
| deletions | `.stats.deletions` |
| files_changed | `.files | length` |

**Cheaper alternative** if you only need weekly totals per developer (skips 1b entirely):

```
GET https://api.github.com/repos/{owner}/{repo}/stats/contributors
```

returns, per author, 52 weekly buckets `{w: <unix week start>, a: additions, d: deletions, c: commits}`.
(First call may return `202 Accepted` while GitHub computes the stats — retry after a few seconds.)

### 1c. Working example

```python
import os, csv, requests

TOKEN = os.environ["GITHUB_TOKEN"]
H = {"Authorization": f"Bearer {TOKEN}", "Accept": "application/vnd.github+json",
     "X-GitHub-Api-Version": "2022-11-28"}
REPOS = ["acme-bank/atlas-core-batch", "acme-bank/atlas-data-ingest", "acme-bank/atlas-risk-engine",
         "acme-bank/atlas-api-gateway", "acme-bank/atlas-recon-tools", "acme-bank/atlas-devops-infra"]
SINCE = "2026-05-13T00:00:00Z"

rows = []
for repo in REPOS:
    page = 1
    while True:
        r = requests.get(f"https://api.github.com/repos/{repo}/commits",
                         headers=H, params={"since": SINCE, "per_page": 100, "page": page})
        r.raise_for_status()
        batch = r.json()
        for c in batch:
            login = (c.get("author") or {}).get("login") or c["commit"]["author"]["email"]
            det = requests.get(c["url"], headers=H).json()          # 1 extra call for stats
            rows.append([c["sha"], repo, login, c["commit"]["author"]["name"],
                         c["commit"]["author"]["date"], c["commit"]["message"].splitlines()[0],
                         det["stats"]["additions"], det["stats"]["deletions"], len(det.get("files", []))])
        if len(batch) < 100: break
        page += 1

with open("data/commits.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["sha","repo","author_login","author_name","commit_date","message","additions","deletions","files_changed"])
    w.writerows(rows)
```

Rate limits: 5,000 requests/hour authenticated. The per-commit stats call is the expensive part
(~1500 commits ≈ 1500 calls); use `stats/contributors` if that's too slow, or drop the
additions/deletions columns — the dashboard tolerates them being 0.

`data/developers.csv` and `data/repos.csv` are reference data you maintain by hand
(login → display name / team mapping), or pull once from
`GET /orgs/{org}/members` and `GET /orgs/{org}/repos`.

---

## 2. Jira — assigned issues + time logged (last 2 weeks)

Jira Cloud REST API v3. Auth is HTTP Basic with email + API token:

```bash
curl -u "$JIRA_EMAIL:$JIRA_TOKEN" -H "Accept: application/json" ...
```

### 2a. Issues assigned to the team → `data/jira_issues.csv`

Columns: `issue_key, url, summary, issue_type, status, priority, assignee_login, assignee_account_id, story_points, created_date, updated_date`

```
GET $JIRA_BASE/rest/api/3/search/jql
  ?jql=assignee in ("aarav.sharma@bank.com", ...) AND statusCategory != Done ORDER BY updated DESC
  &fields=summary,issuetype,status,priority,assignee,created,updated,customfield_10016
  &maxResults=100
```

| CSV column | JSON path (per issue) |
|---|---|
| issue_key | `.key` |
| url | `$JIRA_BASE + "/browse/" + .key` |
| summary | `.fields.summary` |
| issue_type | `.fields.issuetype.name` |
| status | `.fields.status.name` |
| priority | `.fields.priority.name` |
| assignee_account_id | `.fields.assignee.accountId` |
| story_points | `.fields.customfield_10016` (story-points field id varies — check `GET /rest/api/3/field`) |
| created_date / updated_date | `.fields.created` / `.fields.updated` |

`assignee_login` is your own GitHub↔Jira mapping — keep it in `developers.csv`
(`jira_account_id` column) and join on `accountId`.

Pagination: the response returns `nextPageToken`; pass it back as `&nextPageToken=` until absent.
(On Jira Server/DC use `POST /rest/api/2/search` with `startAt`/`total` instead.)

### 2b. Worklogs for the last 2 weeks → `data/jira_worklogs.csv`

Columns: `worklog_id, issue_key, author_login, started_date, time_spent_hours`

Step 1 — find issues that received worklogs from your people recently:

```
jql = worklogAuthor in ("aarav.sharma@bank.com", ...) AND worklogDate >= -14d
GET $JIRA_BASE/rest/api/3/search/jql?jql=...&fields=summary
```

Step 2 — pull each issue's worklogs (1 call per issue):

```
GET $JIRA_BASE/rest/api/3/issue/{issueKey}/worklog?startedAfter=<epoch-millis-14d-ago>
```

| CSV column | JSON path (per worklog) |
|---|---|
| worklog_id | `.id` |
| issue_key | (the issue you queried) |
| author_login | join `.author.accountId` → developers.csv |
| started_date | `.started` (keep date part) |
| time_spent_hours | `.timeSpentSeconds / 3600` |

**Filter again client-side** on `started >= now-14d` and on your authors — the endpoint
returns all of an issue's worklogs if `startedAfter` is omitted, and other people's
worklogs on shared issues either way.

### 2c. Working example

```python
import os, csv, requests, datetime

BASE = os.environ["JIRA_BASE"]
AUTH = (os.environ["JIRA_EMAIL"], os.environ["JIRA_TOKEN"])
TEAM = {"5f8a1c001": "asharma", "5f8a1c002": "jchen"}   # accountId -> github login (from developers.csv)
since = datetime.datetime.now(datetime.timezone.utc) - datetime.timedelta(days=14)
since_ms = int(since.timestamp() * 1000)

jql = f'worklogAuthor in ({",".join(repr(a) for a in TEAM)}) AND worklogDate >= -14d'
r = requests.get(f"{BASE}/rest/api/3/search/jql", auth=AUTH,
                 params={"jql": jql, "fields": "summary", "maxResults": 100})
r.raise_for_status()
issues = [i["key"] for i in r.json()["issues"]]

rows = []
for key in issues:
    wl = requests.get(f"{BASE}/rest/api/3/issue/{key}/worklog", auth=AUTH,
                      params={"startedAfter": since_ms}).json()
    for w in wl["worklogs"]:
        acct = w["author"]["accountId"]
        if acct in TEAM and w["started"][:10] >= since.date().isoformat():
            rows.append([w["id"], key, TEAM[acct], w["started"][:10],
                         round(w["timeSpentSeconds"] / 3600, 2)])

with open("data/jira_worklogs.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["worklog_id","issue_key","author_login","started_date","time_spent_hours"])
    w.writerows(rows)
```

Sanity check the dashboard expects: each developer's 2-week total lands around 40–50 h
(of an 80 h fortnight); the "Capacity used" meters are computed against `CAPACITY_2W = 80`
in `devdashboard.html` — change that constant if your fortnight differs.

---

## 3. Data-inventory dashboard (`datadashboard.html`)

Its four CSVs (`input_data_points.csv`, `output_data_points.csv`, `processes.csv`,
`daily_history.csv`) are registry/telemetry data you own — typically maintained by hand or
exported from your scheduler / file-watcher logs. Column definitions are in the CSV headers;
`daily_history.csv` wants one row per data point per batch date with
`size_mb, record_count, actual_time_ct, status ∈ {on_time, late, missed}`.

## 4. Rebuild after refreshing CSVs

```bash
python3 build_site.py     # re-injects data/*.csv into site/*.html
```

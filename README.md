# Credit Card Spending Analysis

A private, single-file web app for analyzing household credit card spending across American Express, Apple Card, Chase, and manually named cards. Drop in your CSV exports and get an interactive dashboard with categorized transactions, charts, trend analysis, tax summaries, subscription review, and credit card optimization recommendations — all processed locally in your browser with no data ever leaving your device.

Live at **[expenses.jsorlando.com](https://expenses.jsorlando.com)**

---

## What It Does

The app takes CSV transaction exports from one or more credit cards and produces a unified view of household spending. It auto-detects known card formats, asks you to confirm or name each file, categorizes every transaction using a rule-based engine, and presents the data across seven interactive views.

### Upload & Parsing

The landing screen accepts CSV files via drag-and-drop or file picker. Each file is parsed entirely in the browser — no server, no network request for your data. The app identifies the card type automatically by inspecting the CSV header:

| Card | Detection signal |
|------|-----------------|
| American Express | `Card Member` + `Account #` columns |
| Apple Card | `Clearing Date` + `Purchased By` columns |
| Chase | `Post Date` + `Memo` columns |

Payments, credits, and refunds are filtered out. Only charges are included. The dashboard derives its display year from the uploaded transaction data.

### Categorization

Every transaction is run through a priority-ordered set of regex rules that match merchant names, original card-assigned categories, and known merchant patterns. The 23 supported categories are:

| Category | Category | Category |
|----------|----------|----------|
| Dining & Restaurants | Groceries | Shopping & Retail |
| Travel & Transport | Health & Medical | Pet Care |
| Technology | Home & Garden | Beauty & Wellness |
| Entertainment & Sports | Golf | Fitness |
| Dry Cleaning | Coffee | Wine & Spirits |
| Insurance | Charity & Donations | Flowers & Gifts |
| Auto Maintenance | Utilities | Subscriptions |
| Fees & Interest | Other | |

#### Learned Merchant Rules

Any transaction's category can be overridden by clicking its category badge in the Transactions tab. When you reassign a category, the app creates a merchant-level rule that applies to **all** transactions from that merchant — past and future. Learned rules are marked with 🧠 and persist across sessions via `localStorage`. Rules can be removed individually or reset all at once.

#### Other Review

The Transactions tab includes an uncategorized merchant workbench for rows that still land in **Other**. It groups uncategorized transactions by merchant, shows suggested categories from merchant/category hints when available, and applies merchant-level rules in one click.

---

## The Seven Tabs

### Overview

High-level household snapshot:

- **Stat cards** — total spend, active-month average, top category, peak month
- **Donut chart** — top 10 categories by spend (remaining categories collapsed into an "N more" slice)
- **Stacked bar chart** — monthly spend broken out by card, including manually named cards
- **Card totals** — annual spend per card with color-coded badges
- **Cardholder split** — per-person spend with progress bars, shown only when multiple cardholders are present (names pulled directly from the CSV)

### Categories

A card for every category that has at least one transaction:

- Total spend, transaction count, average transaction size
- Progress bar showing share of total household spend
- **Card optimization tip** — the specific credit card and reward rate recommended for that category, with notes on eligibility and trade-offs

### Trends

Time-series and merchant views:

- **Monthly Spending Trend** — same stacked bar chart as Overview, with a monthly summary grid showing each month's total
- **Category heatmap** — month-by-month spend intensity by category
- **Day-of-week chart** — spend totals and transaction counts by weekday
- **Top Merchants by Spend** — horizontal bar chart of the 12 highest-spend merchants, color-coded by category, showing dollar amount and transaction count on hover

### Transactions

Full transaction ledger with:

- **Search** — filters by description or merchant name (real-time)
- **Card filter** — show one card or all
- **Category filter** — show one category or all
- **Sortable columns** — date (default: newest first) and amount
- **Learned-only toggle** — show only transactions with overridden categories
- **Other Review** — bulk-review uncategorized merchant groups and apply suggested merchant rules
- **Pagination** — 50 transactions per page
- **Inline category editing** — click any badge to reassign; changes propagate to all matching merchants instantly

### Insights

Hardcoded analysis cards with reward optimization math specific to the household's actual spend:

- Unrealized reward opportunity across dining, groceries, travel, and subscriptions (estimated value vs. assumed 1% baseline)
- Card fee and interest flagging
- Transit spending patterns
- Recurring charge identification
- Category-specific observations (e.g. high retail spend, pet care costs, health & medical)

### Subscriptions

Recurring-charge review:

- Detects merchants that appear in at least two months
- Estimates monthly and annualized spend
- Lets you tag each recurring merchant as Keep, Review, or Cancel
- Saves tags locally in `localStorage`

### Tax

Tax-prep helper views:

- Charitable contribution summary with $250+ receipt warnings
- Copyable donation ledger for tax prep
- Medical and dental expense summary with a 7.5% AGI threshold calculator

---

## How It's Built

The entire application is a single `.html` file with no build step, no bundler, and no backend. All dependencies are loaded from CDNs at runtime.

### Stack

| Layer | Technology |
|-------|-----------|
| UI framework | [React 18](https://react.dev) (UMD build via unpkg) |
| JSX compilation | [Babel Standalone](https://babeljs.io/docs/babel-standalone) (in-browser transform) |
| Charts | [Chart.js 4.4](https://www.chartjs.org) |
| Styling | Inline CSS + a small set of utility classes (no framework) |
| Persistence | `localStorage` (merchant overrides only) |
| Hosting | AWS S3 + CloudFront + ACM + Route 53 |

### Key React Patterns

- `useState` / `useEffect` / `useMemo` / `useCallback` / `useRef` throughout
- Chart instances are created imperatively via `useRef` and destroyed on cleanup to prevent canvas leaks
- `useMemo` gates all expensive aggregations (category rollups, merchant rollups, filtered transaction lists) behind dependency arrays
- Merchant override state is stored as a flat `{ merchantKey → category }` map and merged into transactions at render time

### CSV Parsing

A hand-rolled CSV parser handles quoted fields and escaped quotes (`""`). Each card format has its own parser function (`parseAmex`, `parseApple`, `parseChase`) that normalizes the output into a shared transaction shape:

```
{
  id, year, month, day, monthKey, dateStr,
  description, merchant, amount,
  card, cardHolder, originalCategory
}
```

### Categorization Engine

Rules are evaluated in priority order — more specific rules (e.g. `Pet Care`, `Golf`) run before broader ones (e.g. `Dining`, `Shopping`). Before categorization, each transaction receives a canonical merchant identity and readable display name, so rules can test the raw statement text, original card category, canonical merchant, and cleaned display name together.

### Merchant Display Names

Raw statement descriptions are preserved, but the dashboard formats noisy merchant text into readable display names for tables, charts, recurring-charge groups, and review workflows. For example, a statement label such as `MERCHANT 555.123.4567NEW YORK NY` can display as `Merchant Name`, while the original text remains available in tooltips and copied tax-prep output.

The app also keeps a separate canonical merchant identity for grouping and learned rules. This lets processor variants collapse into the same merchant even when their statement text differs. Recurring equal-amount charges are a useful signal when deciding which aliases should be merged.

---

## Deployment

Hosted on AWS using a fully private architecture:

```
Browser → Route 53 → CloudFront (HTTPS, ACM cert) → S3 (private bucket, OAC)
```

- **S3 bucket**: private, all public access blocked
- **CloudFront**: PriceClass_100 (US/EU), HTTP → HTTPS redirect
- **ACM certificate**: DNS-validated via Route 53
- **Origin Access Control**: only the CloudFront distribution can read from S3

### Deploying Updates

Pushes to `main` publish automatically through GitHub Actions. The workflow uploads `index.html` to S3 and invalidates CloudFront using GitHub OIDC, so no long-lived AWS access keys are stored in the repo.

Required GitHub repository settings:

| Type | Name | Value |
|------|------|-------|
| Secret | `AWS_ROLE_TO_ASSUME` | ARN of the IAM role trusted by GitHub Actions |
| Variable | `AWS_REGION` | AWS region used for AWS API calls |
| Variable | `S3_BUCKET` | S3 bucket name that backs the site |
| Variable | `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront distribution ID |

The IAM role should be scoped to this repository and only needs permission to upload the single HTML file and create CloudFront invalidations. Example permission policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::<your-bucket>/index.html"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
    }
  ]
}
```

Example trust policy condition for the role:

```json
{
  "StringEquals": {
    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
    "token.actions.githubusercontent.com:sub": "repo:jonathansorlando/spending-analysis:environment:production"
  }
}
```

Manual equivalent:

```bash
# Upload new version
aws s3 cp index.html s3://<your-bucket>/index.html --content-type "text/html"

# Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id <your-distribution-id> --paths "/*"
```

---

## Privacy

All CSV parsing and data processing runs in your browser. No transaction data, file contents, or personal information is sent to any server. The S3 bucket and CloudFront distribution serve only the static `index.html` file itself.

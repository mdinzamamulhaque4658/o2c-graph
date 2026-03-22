# Order to Cash — Graph Intelligence System

A graph-based data exploration and natural language query interface for SAP Order-to-Cash (O2C) process data.

---

## Live Demo

> Add your deployed link here after deployment

---

## What It Does

- Loads all JSONL data files and builds an in-memory relational model
- Visualizes the O2C process as an interactive D3 force-directed graph
- Accepts natural language questions and answers them with SQL logic over the real dataset
- Highlights graph nodes referenced in query results
- Blocks off-topic queries with guardrails

---

## Architecture

```
data/ (JSONL files)
    ↓ fetch + parse in browser
In-Memory Tables (JS objects, keyed by table name)
    ↓
Local Query Engine (pure JS)  ←→  Gemini API (NL → SQL → answer)
    ↓
D3.js Force Graph  +  Chat Interface
```

### Why No Backend?

The entire app is a single `index.html` file — no server, no database process, no build step. This makes it:
- Trivially deployable (drag-and-drop to Vercel / GitHub Pages)
- Instantly reproducible by evaluators
- Zero infrastructure cost

### Data Loading

Files are fetched from `data/` at runtime. Each file is identified by inspecting the keys of its first record and matched to a known table schema. Multiple files with the same schema are merged (e.g. `billing_headers` spans 3 JSONL files).

### Storage / Query Engine

Rather than shipping a full SQLite WASM binary (sql.js is ~1.5MB), the app uses a **pure JavaScript query engine** that:
- Runs all JOINs as `Array.filter()` and `Map` lookups — O(n) per join
- Handles the 8 most common O2C query patterns locally with zero latency
- Falls back to Gemini for arbitrary natural language questions

For the graph itself, nodes are capped at a representative sample (customers, top 20 products, all deliveries/billings/journals) to keep D3 simulation performant in the browser.

### LLM Prompting Strategy

The system prompt passed to Gemini does the following:

1. **Schema context** — all 13 table definitions with column names and data types
2. **Join paths** — explicit mapping of foreign keys (e.g. `billing_items.referenceSdDocument → delivery_headers.deliveryDocument`)
3. **O2C flow description** — `Sales Order → Delivery → Billing → Journal Entry → Payment`
4. **Output format** — forces JSON response with `{ sql, answer, type, columns, rows }` for deterministic parsing
5. **Temperature = 0.1** — near-deterministic SQL generation

### Guardrails

Two layers prevent off-topic use:

1. **Client-side keyword filter** — blocks questions containing "weather", "recipe", "capital of", "sports", "movie", "bitcoin", etc. before any API call is made
2. **System prompt instruction** — Gemini is explicitly told to respond with `type: "offtopic"` and a canned message for anything not related to the O2C dataset. This is stated as a strict rule number 1 in the prompt.

---

## Key Join Map

| From | Via | To |
|---|---|---|
| `billing_items.referenceSdDocument` | = | `delivery_headers.deliveryDocument` |
| `delivery_items.referenceSdDocument` | = | `sales_order_items.salesOrder` |
| `billing_headers.accountingDocument` | = | `journal_entries.accountingDocument` |
| `journal_entries.referenceDocument` | = | `billing_headers.billingDocument` |
| `journal_entries.accountingDocument` | = | `payments.accountingDocument` |
| `billing_headers.soldToParty` | = | `business_partners.customer` |
| `delivery_items.plant` | = | `plants.plant` |

---

## Dataset

| Table | Records | Files |
|---|---|---|
| billing_headers | 243 | 3 |
| billing_items | 245 | 2 |
| delivery_headers | 86 | 1 |
| delivery_items | 137 | 2 |
| sales_order_items | 167 | 2 |
| journal_entries | 123 | 4 |
| payments | 120 | 1 |
| business_partners | 8 | 1 |
| schedule_lines | 179 | 2 |
| products | 69 | 2 |
| product_descriptions | 69 | 2 |
| plants | 44 | 1 |
| product_plants | 3,036 | 4 |
| product_storage_locations | ~14,000 | 18 |

---

## Graph Model

**Node types (7):** Sales Order, Delivery, Billing Document, Journal Entry, Payment, Customer, Product, Plant

**Edge types (6):**
- CUSTOMER to BILLING (INVOICED)
- BILLING to DELIVERY (BILLED_FROM)
- DELIVERY to SALES ORDER (FULFILLS)
- DELIVERY to PLANT (FROM_PLANT)
- BILLING to JOURNAL (POSTED_TO)
- JOURNAL to PAYMENT (CLEARED_BY)

---

## Example Queries Supported

```
Which products appear in the most billing documents?
Trace the full flow of billing document 90504248
Find sales orders with broken or incomplete flows
Which customers have the highest outstanding amounts?
Show delivery status summary
List all customers
Show cancelled billing documents
Revenue and payment summary
```

---

## Setup and Run Locally

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/o2c-graph
cd o2c-graph

# Serve (required — file:// protocol will not work due to fetch())
npx serve .
# or
python3 -m http.server 8080

# Open browser at:
# http://localhost:3000  (npx serve)
# http://localhost:8080  (python)
```

Then enter your Google Gemini API key when prompted.
Get one free at https://aistudio.google.com/app/apikey

---

## Deployment on Vercel

```bash
npm i -g vercel
vercel
# Follow prompts to get a live URL
```

---

## Project Structure

```
o2c-graph/
├── index.html        # Entire app (self-contained)
├── data/             # All JSONL data files (49 files)
│   └── *.jsonl
├── sessions/         # AI coding session logs
│   └── claude_session.md
└── README.md
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Graph visualization | D3.js v7 force-directed |
| Query engine | Pure JavaScript array joins |
| LLM | Google Gemini 1.5 Flash free tier |
| Hosting | Vercel or GitHub Pages |
| Build system | None — single HTML file |

---

## AI Tools Used

Built with Claude (Anthropic) via claude.ai. Session logs are in the sessions/ folder.

# Cribl Search — Explained Slide by Slide

## Slide 1: Title
**Cribl Search**
*Federated Search, Notebooks, and Data Lifecycle — Explained Simply*

Sets up the three themes the deck will cover: Architecture, Federated Search, and Lake/Retention/Lifecycle.

---

## Slide 2: What We'll Cover
A five-part roadmap for the presentation:

1. **Cribl Search — the big idea** — why search doesn't require moving your data first
2. **Federated search across data-in-place** — querying data where it already lives
3. **Architecture walkthrough** — how a search request actually flows
4. **Notebooks** — an interactive workspace for exploring results
5. **Cribl Lake, retention & lifecycle** — where data rests, and how long it stays

---

## Slide 3: The Big Idea
**Core message:** Search the data where it already lives — nothing to move, nothing to duplicate.

- **Old way:** Traditional search tools require you to first collect and index your data (`Data → Index/Store → then search`).
- **Cribl Search way:** Flips this model — it sends the *query* out to the *data*, instead of bringing the data to the query (`Query → Data, in place`).
- **What this enables:** You can search logs, metrics, and events sitting in object storage, other platforms, or Cribl Lake — with no separate ingest/indexing step.
- **Payoff:** Faster time-to-answer, lower storage cost, and access to data you may not have wanted to index at all.

---

## Slide 4: Federated Search Across Data-in-Place
**Core message:** One query, many data stores — each searched where it sits.

Data sources shown feeding into Cribl Search:
- Cribl Lake
- Object Storage (S3 / GCS / Azure)
- Splunk
- Other Platforms

Cribl Search queries each store directly, in its native location, and combines everything into **Unified Results** — one result set across every source. This pushes the work down to the source and only pulls back what's needed, reducing data movement and cost.

---

## Slide 5: Architecture Walkthrough
**Core message:** How a search request flows through the system, in four steps:

1. **You ask** — type a query in the Search UI or Notebook
2. **Search plans** — Cribl Search figures out which datasets to touch
3. **Work is pushed down** — each connected source runs its portion of the query
4. **Results return** — partial results are merged into one answer

**Components involved:**
| Component | Role |
|---|---|
| Search UI / Notebook | Where you write and run queries |
| Search Coordinator | Plans the query and merges results |
| Connected Datasets | Cribl Lake, object storage, other platforms |

---

## Slide 6: Notebooks
**Core message:** An interactive workspace built around your search queries.

- A **Notebook** is a saved, shareable page made up of **cells** — each cell holds a query, a note, or a visualization.
- Cells can be run one at a time, in any order, to explore a problem step by step instead of writing one giant query.
- You can mix findings with plain-text notes, so the investigation and the explanation live in the same place.
- Notebooks can be saved and shared so a teammate can re-run the same steps or continue the investigation.

**Example shown:** "Investigation: Login Failures" — a note describing the check, a query (`search source=auth status=failed`), and a resulting chart of failures by hour.

---

## Slide 7: Cribl Lake
**Core message:** A managed storage layer for the data you collect.

- Cribl Lake is **object-storage-backed** — cheap, durable storage without running your own infrastructure.
- Data lands in Lake in an **open format**, organized into **datasets**, so it stays queryable by Cribl Search and other tools.
- It's a natural landing zone for data you want to keep but don't need in an expensive, always-hot index.
- Because Cribl Search can query Lake directly, data goes from "collected" to "searchable" without a separate loading step.

**Flow shown:** Data Sources (logs, metrics, events) → Cribl Lake (object storage, open format, datasets) → Cribl Search (query it directly).

---

## Slide 8: Retention & Lifecycle Policies
**Core message:** Rules that decide how long data stays, and what happens to it over time.

**Lifecycle stages:**
1. **Ingested** — data lands in Cribl Lake
2. **Active retention** — data is kept for a set period, fully searchable
3. **Lifecycle rule triggers** — at a defined age, a policy takes action
4. **Expired / removed** — data older than the retention window is deleted

**Key concepts:**
| Concept | Explanation |
|---|---|
| Retention period | How long a dataset is kept before it's eligible for removal — set per dataset based on compliance/operational needs |
| Lifecycle policy | A rule that acts automatically once data reaches a certain age (e.g., delete data older than 90 days) |
| Storage cost control | Because Lake storage is inexpensive, retention windows can be generous — but policies still keep storage from growing unbounded |

---

## Slide 9: Putting It Together
A recap of all five concepts in one place:

- **Cribl Search** — Query data directly, wherever it lives — no mandatory index step first.
- **Federated search** — One query fans out across multiple data-in-place sources and returns one unified answer.
- **Architecture** — Search UI/Notebook → Coordinator plans the query → work is pushed to each source → results merge.
- **Notebooks** — A cell-based workspace for step-by-step investigation, mixing notes, queries, and results.
- **Cribl Lake** — Cost-effective object storage for datasets, queryable in place by Cribl Search.
- **Retention & lifecycle** — Policies define how long data stays and automatically remove it once it ages out.

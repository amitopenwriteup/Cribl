# Cribl Search Sandbox Lab — Exploring Search Engines and KQL

## Lab Duration

**60–90 minutes**

## Lab Environment

Use the existing Cribl.Cloud sandbox shown in the lab screenshot.

You should have access to:

* **Search Home**
* **History**
* **Saved Searches**
* **Dashboards**
* **Notebooks**
* **Data**
* **Knowledge**
* **Packs**

The sandbox already contains the sample Dataset:

`cribl_search_sample`

It also contains several `cribl_edge_*` Datasets.

---

# Lab Objectives

By the end of this lab, you will be able to:

1. Navigate Cribl Search.
2. Identify Search Datasets.
3. Write basic KQL queries.
4. Use `limit`, filtering, `summarize`, and `timestats`.
5. Identify fields in network-flow data.
6. Understand the relationship between a Dataset and an Engine.
7. Inspect the Engine configuration in **Data → Engines**.
8. Explain the difference between a **Lakehouse Engine** and a **Federated Engine**.
9. Understand how engine sizing relates to ingest and search capacity.
10. Design an engine/Dataset architecture for a realistic workload.

Cribl describes Lakehouse Engines as storage-plus-compute resources for data ingested into Cribl Search, while the Federated Engine provides compute for searching data that remains in external systems.

---

# Part 1 — Explore the Search Home Screen

## Step 1 — Open Search

You are already on:

**Products → Search → Search Home**

Your screen should look similar to the supplied screenshot.

Identify these areas:

### A. Query editor

At the top:

`Enter search terms here or check our documentation`

This is where you enter KQL queries.

### B. Time range

Your sandbox currently shows:

**1 hour ago**

and:

**Local (GMT+5:30)**

### C. Available Datasets

The left panel contains Datasets such as:

* `cribl_edge_appscope_events`
* `cribl_edge_appscope_metrics`
* `cribl_edge_kubernetes_logs`
* `cribl_edge_logs`
* `cribl_edge_metrics`
* `cribl_edge_prometheus_scraper`
* `cribl_edge_spool`
* `cribl_edge_state`
* `cribl_edge_system_logs`

### D. Sample Searches

The right panel contains ready-made examples.

---

# Exercise 1 — Find the Sample Dataset

In **Available Datasets**, search/filter for:

`cribl_search_sample`

If it is available, select it.

If it isn't visible, use the Dataset search/filter box.

### Question

What is the Dataset ID?

**Answer:**

`cribl_search_sample`

Cribl's getting-started documentation uses this built-in Dataset for learning Search.

---

# Part 2 — Your First Search

In the query editor, enter:

```text
dataset="cribl_search_sample"
| limit 100
```

Click **Search**.

## Observe

You should receive up to 100 events.

Look at the returned fields.

Pay particular attention to fields such as:

* `_time`
* `_raw`
* `srcaddr`
* `dstaddr`
* `srcport`
* `dstport`
* `action`

The `limit` operator is useful when first exploring an unfamiliar Dataset because it prevents you from immediately requesting an unnecessarily large result set.

---

# Exercise 2 — Inspect the Data

Pick one event.

Record:

| Field     | Your observation |
| --------- | ---------------- |
| `_time`   |                  |
| `srcaddr` |                  |
| `dstaddr` |                  |
| `srcport` |                  |
| `dstport` |                  |
| `action`  |                  |

### Question

What type of telemetry does this Dataset appear to contain?

**Expected answer:** Network/VPC flow-style telemetry.

---

# Part 3 — Count Events

Replace the query with:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count()
```

Run it.

### Question

Approximately how many events were returned?

Record your result:

**Event count:** __________

---

# Part 4 — Group Events by Source

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by source
```

Cribl uses `summarize count()` to aggregate events into groups.

## Observe

The results should now be aggregated rather than displaying individual events.

### Questions

1. How many different `source` values do you see?
2. Which source has the largest count?
3. What is the count?

Record:

**Largest source:** __________

**Count:** __________

---

# Part 5 — Explore Network Traffic

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by srcaddr
```

This gives you a count of events grouped by source IP address.

### Challenge

Modify the query to group by:

`dstaddr`

Expected structure:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by dstaddr
```

### Questions

1. Which destination receives the most events?
2. How many events does it receive?
3. Does the result look different from the `srcaddr` aggregation?

---

# Part 6 — Investigate ACCEPT vs REJECT

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by action
```

You should see traffic grouped according to the `action` field.

Now try:

```text
dataset="cribl_search_sample"
| where action has_any ("ACCEPT", "REJECT")
| summarize count() by action
```

The `has_any` operator can be used to match multiple values in a field.

### Questions

* How many `ACCEPT` events?
* How many `REJECT` events?
* Which is more common?
* What percentage of the observed events are rejected?

Calculate:

`REJECT / total × 100`

**Your result:** __________ %

---

# Part 7 — Explore Ports

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by dstport
```

### Questions

1. What are the most common destination ports?
2. Do you recognize any of them?
3. What services might commonly use those ports?

### Challenge

Modify the query so that you group by both destination address and destination port:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by dstaddr, dstport
```

Cribl supports multiple grouping fields in `summarize`.

---

# Part 8 — Create a Time-Series Search

Now explore traffic over time.

Try:

```text
dataset="cribl_search_sample"
| limit 1000
| timestats span=1m
```

The `timestats` operator is designed for time-series aggregation.

Look at the **Chart** view.

### Questions

1. Does the traffic appear constant?
2. Are there peaks?
3. Are there periods with very little activity?

---

# Part 9 — Explore the Time Picker

At the top of the screen, change:

**1 hour ago**

to another available time range.

For example:

* 15 minutes
* 30 minutes
* 6 hours
* 24 hours

Run the same query again.

### Question

How does changing the time range affect the result?

Explain:

> Why is restricting the time range important when investigating large Datasets?

Cribl recommends constraining initial searches with a `limit`, count, or time window when you don't yet know how large the Dataset is.

---

# Part 10 — Now Explore Engines

This is where the lab connects your hands-on Search experience to the **Cribl Search Engines** documentation.

From the left navigation, select:

**Data**

Then find:

**Engines**

You are now looking at the infrastructure that powers Search.

## Exercise 3 — Identify the Engines

Record what you see.

| Item                 | Your sandbox |
| -------------------- | ------------ |
| Lakehouse Engines    |              |
| Lakehouse Engine IDs |              |
| Engine sizes         |              |
| Engine status        |              |
| Federated Engine     |              |
| Federated size       |              |

Do not change anything yet.

---

# Part 11 — Understand What You Are Looking At

Cribl currently describes two engine types.

## Lakehouse Engine

A Lakehouse Engine provides:

**Storage + Compute**

It stores data ingested into Cribl Search and powers searches against that data.

Multiple Lakehouse Engines can exist in a Workspace.

Think:

```text
Source
   |
   v
Lakehouse Engine
   |
   +---- Dataset A
   |
   +---- Dataset B
   |
   +---- Dataset C
   |
   v
Search
```

## Federated Engine

The Federated Engine provides:

**Compute**

The data remains in an external system such as S3, Blob Storage, or another supported external source.

Think:

```text
External Storage
       |
       |  data stays here
       v
Federated Engine
       |
       v
    Search
```

A Workspace has a single Federated Engine, while multiple Lakehouse Engines can be provisioned.

---

# Exercise 4 — Architecture Test

For each scenario, choose:

**Lakehouse** or **Federated**

### Scenario 1

You want to ingest firewall logs into Cribl Search and retain them for 30 days.

Answer: __________

### Scenario 2

You have 10 TB of historical logs in Amazon S3 and want to search them without copying them into Cribl Search.

Answer: __________

### Scenario 3

You need several Search Datasets with different retention periods.

Answer: __________

### Scenario 4

You want to search external data in place.

Answer: __________

### Scenario 5

You want Cribl Search to store and accelerate recently ingested data.

Answer: __________

---

# Part 12 — Engine Capacity Exercise

Go back to:

**Data → Engines**

Look at your Lakehouse Engine size.

The important sizing concept is:

> Lakehouse engine size is based on the amount of **uncompressed data ingested per day** that the engine is designed to handle.

Retention is configured separately on the Dataset.

### Scenario

Your company produces:

```text
Firewall       200 GB/day
Application    300 GB/day
Authentication 100 GB/day
---------------------------
Total          600 GB/day
```

### Question

What capacity would you plan for?

Don't simply choose exactly 600 GB/day.

Consider:

* growth
* traffic spikes
* additional Sources
* future Datasets
* operational headroom

---

# Part 13 — Dataset vs Engine

This is an important concept.

A Dataset controls things such as:

* data organization
* retention
* search scope

The Engine provides:

* storage capacity
* compute capacity

Cribl allows each Search Dataset to have its own retention period, from 1 day to 10 years.

## Scenario

You have:

```text
Security Logs       → 30 days
Application Logs    → 7 days
Audit Logs          → 365 days
```

### Question

Do you necessarily need three Lakehouse Engines?

**No.**

You can use appropriate Search Datasets and configure their retention independently, provided the engine capacity is appropriate for the workload.

---

# Part 14 — Design Your Sandbox Architecture

Imagine your sandbox receives:

```text
100 GB/day firewall logs
200 GB/day application logs
50 GB/day authentication logs
```

Design an environment.

Complete:

```text
Lakehouse Engine(s):
____________________

Estimated ingest:
____________________

Firewall Dataset:
____________________

Firewall retention:
____________________

Application Dataset:
____________________

Application retention:
____________________

Authentication Dataset:
____________________

Authentication retention:
____________________
```

### Architecture Question

Would you use:

**Option A**

```text
One Lakehouse Engine
       |
       +--- Firewall Dataset
       +--- Application Dataset
       +--- Authentication Dataset
```

or:

**Option B**

```text
Engine 1
   |
   +--- Firewall

Engine 2
   |
   +--- Application

Engine 3
   |
   +--- Authentication
```

Explain why.

---

# Part 15 — Optional Hands-On: Resize Experiment

Only perform this exercise if your sandbox permissions allow it and the environment is explicitly intended for configuration changes.

Go to:

**Data → Engines**

Select the Lab/Lakehouse Engine.

Record:

**Current size:** __________

**Current status:** __________

If resizing is available, select another size.

Record the status transitions.

### Questions

1. What status appears during resizing?
2. What status indicates the engine is operational?
3. Does resizing the Federated Engine resize a Lakehouse Engine?
4. Does changing Dataset retention resize an Engine?

Cribl documents Lakehouse engine sizing and Federated engine sizing as separate capacity decisions. Changing Federated size does not resize Lakehouse Engines.

---

# Part 16 — Federated Search Thought Experiment

Imagine that your organization has:

```text
Amazon S3
|
+-- firewall/
+-- cloudtrail/
+-- application/
+-- archive/
```

The data must remain in S3.

You want analysts to search it from Cribl Search.

### Questions

1. Should you use a Lakehouse Engine or Federated Engine?
2. Where does the data remain?
3. Does the Federated Engine store the data?
4. How many Federated Engines can the Workspace have?
5. What happens if concurrent federated searches exceed capacity?

Cribl states that searches exceeding the Federated Engine's concurrency capacity queue until capacity becomes available.

---

# Part 17 — Capstone Investigation

Return to **Search Home**.

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by action
```

Then investigate the result.

Answer:

### Question 1

Which action is most common?

### Question 2

What are the top destination ports?

Run:

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by dstport
```

### Question 3

What are the busiest destination IP addresses?

```text
dataset="cribl_search_sample"
| limit 1000
| summarize count() by dstaddr
```

### Question 4

What is the relationship between source IP, destination IP, destination port, and action?

### Question 5

If this data were continuously ingested into Cribl Search, which type of Engine would store it?

### Question 6

Where would you configure how long the data is retained?

---

# Final Challenge

You are now the Cribl Search architect.

A SOC generates:

```text
Firewall logs:        400 GB/day
Application logs:     300 GB/day
Authentication logs:  100 GB/day
```

Requirements:

```text
Firewall retention:       30 days
Application retention:    14 days
Authentication retention: 365 days

Historical archive:       20 TB in S3
Analysts:                 15
```

Design the solution.

Your answer must include:

1. Lakehouse Engine strategy.
2. Search Dataset strategy.
3. Retention strategy.
4. Federated Engine strategy.
5. Where the 20 TB archive should remain.
6. How you would handle future ingest growth.
7. How you would handle increased concurrent federated searches.

---

# Knowledge Check

Before finishing the lab, answer these from memory.

1. What does a Lakehouse Engine provide?

2. What does a Federated Engine provide?

3. Can a Workspace have multiple Lakehouse Engines?

4. Can a Workspace have multiple Federated Engines?

5. Where does a Lakehouse Dataset's data live?

6. Where does a Federated Dataset's data live?

7. What determines Lakehouse engine ingest capacity?

8. Is Lakehouse ingest sizing based on compressed or uncompressed data?

9. Where is Dataset retention configured?

10. What happens when federated search concurrency exceeds capacity?

---

# Expected Core Answers

1. **Lakehouse = storage + compute.**

2. **Federated = compute for search-in-place.**

3. **Yes**, multiple Lakehouse Engines can exist in a Workspace.

4. **No**, a Workspace has one Federated Engine.

5. Lakehouse data is stored in Cribl Search/Lakehouse infrastructure.

6. Federated data remains in the external system.

7. Lakehouse engine size is tied to the daily uncompressed ingest capacity.

8. **Uncompressed** ingest.

9. Retention is configured per Search Dataset.

10. Searches queue until Federated Engine capacity becomes available.

These distinctions are the central concepts in the current Cribl Search Engines documentation.

---

# Optional Extension

After completing the lab, explore:

**Data → Datasets**

and compare the Dataset configuration with:

**Data → Engines**

Ask yourself:

> "Is this setting describing the data, or is it describing the compute/storage resource that handles the data?"

That distinction is one of the most useful ways to understand Cribl Search architecture.

Cribl's documentation describes Search Datasets as containers for related events and notes that their retention is independently configurable.

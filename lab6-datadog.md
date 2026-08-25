# Cribl Stream Integration Guide: Datadog

This guide covers connecting Cribl Stream to **Datadog** as a destination — authentication, connection setup, and end-to-end validation.

---

## 1. Overview

```
Sources ──▶ Cribl Stream (Routes / Pipelines) ──▶ Datadog (Logs / Metrics API)
```

The **Datadog Destination** in Cribl Stream sends logs and metrics to Datadog's API. It can also receive Cribl Stream's own internal metrics for monitoring the pipeline itself.

---

## 2. Prerequisites

- A Datadog account (trial or paid)
- Org Owner or API-key-management permissions in Datadog
- Network path from your Cribl Workers to your Datadog site's API endpoint

---

## 3. Get a Datadog API Key

1. In Datadog, go to **Organization Settings > API Keys**.
2. Click **New Key**, name it (e.g. `cribl-stream`).
3. Copy the key — you'll enter it in Cribl.

---

## 4. Configure the Datadog Destination in Cribl Stream

1. Go to **Data > Destinations** (or **Routing > QuickConnect > Add Destination**).
2. Select **Datadog**, then **Add Destination**.
3. Configure under **General Settings**:
   - **Output ID** – unique name, e.g. `datadog-prod`
   - **Authentication method**:
     - **Manual** – paste the API key directly
     - **Secret** – reference a stored secret (recommended for production)
4. Under **Optional Settings**:
   - **Datadog site** – defaults to **US**; other options are US3, US5, Europe, US1-FED, AP1, or Custom (for a manually entered region URL)
   - **Message Field** – **important**: if left blank, Cribl wraps the entire event as a JSON string inside `message`, which breaks Datadog's log parsing, dashboards, and detection rules. Set this to `_raw` (or your log body field) so Datadog receives a clean log line.
   - **Source / Host / Tags / Severity** – Datadog uses these fields to enrich and facet searches. Cribl merges its own tags with any `ddtags` on the event (event tags don't override UI-configured tags).
   - **Allow API key from events** – enable only if some events carry their own `__agent_api_key` (e.g. multiple Datadog Agent sources with different keys feeding the same pipeline).
5. Click **Save**, then **Commit & Deploy**.

### 4.1 Optional: Sending Cribl's own internal metrics to Datadog

Useful for monitoring Cribl Stream's health from within Datadog:

1. In Cribl, go to **QuickConnect**, click **+Add Source**.
2. Under **System Internal**, hover over **Cribl Internal**, choose **Select Existing**, and enable both **CriblLogs** and **CriblMetrics**.
3. Click **+Add Destination**, scroll to the **Datadog** tile, click **+Add New**.
4. Name it (e.g. `Cribl_Datadog`), enter your API key and Datadog site.
5. Both the internal source and destination must use **QuickConnect**, not Routes.

---

## 5. Validating End-to-End Data Flow

### 5.1 Destination-level test

- Open the Datadog destination config, go to the **Test** tab, click **Run Test**. You should see a **Success** message.

### 5.2 Verify in Datadog

1. In Datadog, go to **Logs > Log Explorer**.
2. Filter by `source:<your_source_value>` (whatever you configured in the destination's Source field).
3. Open a log entry and confirm:
   - The `message` field contains the raw log text — **not** a JSON blob wrapping the whole event (a sign the Message Field setting is blank)
   - Tags, host, and source are populated as expected
   - If a Datadog log pipeline is applied, confirm parsed attributes appear correctly

### 5.3 Metrics validation

- If sending Cribl internal metrics, check the Datadog dashboard/metrics explorer for `cribl.*` metrics (events per second, bytes per second, input/output types).

### 5.4 Full pipeline check

- Route a live Source through Cribl Stream to the Datadog destination.
- Compare event counts in Datadog against Cribl Stream's **Monitoring** dashboard (events in/out per destination) to confirm no drops.
- Check the destination's **persistent queue (PQ)** status — a growing PQ signals backpressure or connectivity issues to Datadog's API.

---

## 6. Troubleshooting Checklist

| Symptom | Likely Cause |
|---|---|
| Destination test fails | Wrong Datadog site selected, invalid/revoked API key, firewall blocking outbound HTTPS |
| Logs appear but don't parse / dashboards empty | Message Field left blank — event wrapped as JSON string in `message` |
| "API key invalid, dropping transaction" (Agent-sourced data) | `use_v2_api.series` misconfigured in `datadog.yaml`, or wrong key referenced |
| Metrics missing from Datadog | Internal metrics not routed via QuickConnect, or CriblMetrics source not enabled |
| Intermittent data loss | PQ disk full, backpressure set to "drop", API rate limiting |

---

*Compiled from Cribl and Datadog documentation as of August 2026. Steps may vary slightly by Cribl Stream version.*

# Lab Guide: Configuring Inbound Sources in Cribl Stream
## Syslog and HTTP/S

This lab walks through configuring two inbound Push Sources in Cribl Stream — **Syslog** and **HTTP/S (Bulk API)** — sending test data into each, and verifying the events arrive correctly. It is based on Cribl's official documentation:

- Sources Overview — https://docs.cribl.io/stream/sources/
- Syslog Source — https://docs.cribl.io/stream/sources-syslog/
- Syslog Best Practices — https://docs.cribl.io/stream/4.9/usecase-syslog/
- HTTP/S (Bulk API) Source — https://docs.cribl.io/stream/sources-https/
- Persistent Queues — https://docs.cribl.io/stream/persistent-queues/

---

## Lab Objectives

By the end of this lab, you will be able to:

1. Create and enable a **Syslog Source** listening on UDP and TCP.
2. Send test syslog messages and confirm ingestion using Capture / Live Data.
3. Create and enable an **HTTP Source** and send events via the Cribl Bulk API and Splunk HEC-style endpoint.
4. Route both Sources to a Destination (`devnull` or a test Destination) via QuickConnect.
5. Enable Persistent Queueing on one Source for durability.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| A running Cribl Stream instance | Single-instance, distributed, or Cribl.Cloud (Worker Group access) |
| Admin access to the Cribl Stream UI | To create Sources, Destinations, and Routes |
| A terminal with `curl` and `logger` (or equivalent) | Used to generate test traffic |
| Open network path to the Worker | Firewall rules allowing the ports used below |
| (Optional) `openssl` | If testing TLS-enabled Syslog |

> **Tip:** If you're on Cribl.Cloud with Cribl-managed Workers, use the pre-provisioned Cribl.Cloud endpoints/ports shown on the Data Sources tab instead of arbitrary custom ports.

---

## Part 1 — Syslog Source

### 1.1 Background

Cribl Stream supports receiving syslog data formatted per **RFC 3164** or **RFC 5424**, including message-length-prefixed framing (RFC 5425 / RFC 6587). Cribl Stream ships with a default Syslog Source preconfigured to listen on:

- UDP **9514**
- TCP **9514**
- TCP + TLS **6514**

**Type:** Push | **TLS Support:** Yes | **Event Breaker Support:** No

### 1.2 Create the Syslog Source

1. From the top nav, click **Manage**, then select a Worker Group.
2. Go to **Data > Sources** (or **Routing > QuickConnect** for the graphical flow).
3. From the Sources tiles or left nav, select **Push > Syslog**.
4. You can either:
   - Modify/enable the shipped default (`in_syslog`), or
   - Click **New Source** to create your own.
5. Configure the following fields:

   | Field | Value for this lab |
   |---|---|
   | Input ID | `lab_syslog` |
   | Address | `0.0.0.0` |
   | UDP port | `9514` |
   | TCP port | `9514` |
   | Enabled | `Yes` |

6. Click **Save**.

> **Production note:** enable TLS on the TCP listener if senders will cross the public internet. This lab uses plaintext for simplicity.

### 1.3 Route the Source (QuickConnect)

1. Open **Routing > QuickConnect**.
2. Click **+ Add Source**, select `lab_syslog`.
3. Click **+ Add Destination**, select **Default** (or **DevNull** for a pure ingestion test).
4. Click **Save**, then **Commit and Deploy**.

### 1.4 Send Test Data

From a terminal that can reach the Worker on port 9514:

**UDP test (using `logger`):**
```bash
logger -n <worker-host> -P 9514 -d "Lab test syslog message over UDP"
```

**TCP test (using `logger`):**
```bash
logger -n <worker-host> -P 9514 -T "Lab test syslog message over TCP"
```

**Raw test (using `nc`), RFC 3164-style:**
```bash
echo '<134>Aug 27 10:00:00 labhost testapp: Hello from the Cribl Stream lab' | nc -u <worker-host> 9514
```

Replace `<worker-host>` with your Worker's hostname or IP.

### 1.5 Verify Ingestion

1. In the Cribl Stream UI, go to **Data > Sources**, click into `lab_syslog`.
2. Click **Live Data > Start Capture**.
3. Re-run one of the test commands from 1.4.
4. Confirm the event appears with parsed fields:
   - `_time`, `host`, `appname`, `facility`, `facilityName`, `severity`, `severityName`, `message`

### 1.6 Expected Result

You should see one event per test message, with `host` matching the sending machine and `message` containing your test string. If nothing appears:

- Confirm the firewall allows inbound UDP/TCP 9514 to the Worker.
- Confirm the Source shows **Enabled** and the Route/QuickConnect was committed and deployed.
- Check **Data > Sources > lab_syslog > Logs** for bind errors (e.g., port already in use).

---

## Part 2 — HTTP/S Source

### 2.1 Background

Cribl Stream ships with an **HTTP Source** preconfigured on port **10080**, supporting three built-in endpoint styles on one listener:

- Cribl Bulk API (default path faked as `/cribl`)
- Splunk HEC-style: `/services/collector` or `/services/collector/event`
- Elastic Bulk API: `/elastic` (Cribl appends `/_bulk`)

**Type:** Push | **TLS Support:** Yes | **Event Breaker Support:** No | Supports gzip via `Content-Encoding: gzip`

### 2.2 Create the HTTP Source

1. Go to **Data > Sources**, select **Push > HTTP**.
2. Modify/enable the shipped default, or click **New Source**.
3. Configure the following fields:

   | Field | Value for this lab |
   |---|---|
   | Input ID | `lab_http` |
   | Address | `0.0.0.0` |
   | Port | `10080` |
   | Auth tokens | Leave blank for lab (open access) — generate one for production |
   | Splunk HEC path | Enabled, default `/services/collector` |
   | Enabled | `Yes` |

4. Click **Save**.

### 2.3 Route the Source (QuickConnect)

1. Open **Routing > QuickConnect**.
2. Click **+ Add Source**, select `lab_http`.
3. Click **+ Add Destination**, select **Default** (or **DevNull**).
4. **Save**, then **Commit and Deploy**.

### 2.4 Send Test Data

**Splunk HEC-style event:**
```bash
curl -k https://<worker-host>:10080/services/collector/event \
  -H "Authorization: Splunk <auth-token-if-set>" \
  -d '{"event": "Lab test HEC event", "sourcetype": "lab:http"}'
```

**Cribl Bulk API (raw JSON events, newline-delimited):**
```bash
curl -k https://<worker-host>:10080/cribl \
  -H "Content-Type: application/json" \
  -d '{"message": "Lab test Cribl bulk event", "source": "lab"}'
```

**Gzip-compressed payload:**
```bash
echo '{"event": "Lab test gzip event"}' | gzip > payload.json.gz
curl -k https://<worker-host>:10080/services/collector/event \
  -H "Content-Encoding: gzip" \
  --data-binary @payload.json.gz
```

> If you left **Auth tokens** blank, omit the `Authorization` header. If HTTPS isn't enabled on the Source yet, use `http://` instead of `https://` and drop `-k`.

### 2.5 Verify Ingestion

1. Go to **Data > Sources**, click into `lab_http`.
2. Click **Live Data > Start Capture**.
3. Re-run one of the `curl` commands from 2.4.
4. Confirm the event body and any HEC/Bulk metadata fields appear as expected.

### 2.6 Expected Result

A `200 OK` (or equivalent success) response from `curl`, and a matching event visible in Live Data. Common issues:

- **Connection refused** — Source not enabled, or Route/QuickConnect not deployed.
- **404 on `/services/collector`** — Splunk HEC endpoint not enabled on this Source.
- **401/403** — an Auth token is configured and required in the `Authorization` header.

---

## Part 3 — Add Persistent Queueing (Optional but Recommended)

Persistent Queues (PQ) protect events from loss if the downstream pipeline or Destination is unavailable, by writing them to local disk until the receiver recovers.

1. Open `lab_syslog` (or `lab_http`) in **Data > Sources**.
2. Select the **Persistent Queue Settings** tab.
3. Toggle **Enable Persistent Queue** to **Yes**.
4. On hybrid/on-prem Workers, additionally set:
   - **Max queue size**
   - **Compression**
   - **Queue-full fallback behavior**
5. Click **Save**, then **Commit and Deploy**.

> On Cribl-managed Cloud Workers, this tab only exposes the on/off toggle; PQ is auto-configured at a 1 GB cap per Worker Process.

**Test it:** temporarily disable your Destination (or point it at an unreachable host), send a few test events, then re-enable the Destination and confirm the queued events are delivered once connectivity resumes.

---

## Cleanup

1. In **Routing > QuickConnect**, remove the test Routes for `lab_syslog` and `lab_http` (or leave them if you plan to keep testing).
2. Disable or delete the `lab_syslog` and `lab_http` Source definitions if no longer needed.
3. **Commit and Deploy** to apply the cleanup.

---

## Summary Table

| Source | Default Port(s) | Test Tool | Endpoint / Method |
|---|---|---|---|
| Syslog | UDP 9514, TCP 9514, TCP+TLS 6514 | `logger`, `nc` | Raw syslog over UDP/TCP |
| HTTP/S | 10080 | `curl` | `/services/collector`, `/cribl`, `/elastic/_bulk` |

## Reference Links

- Sources Overview: https://docs.cribl.io/stream/sources/
- Syslog Source: https://docs.cribl.io/stream/sources-syslog/
- Syslog Best Practices: https://docs.cribl.io/stream/4.9/usecase-syslog/
- HTTP/S (Bulk API) Source: https://docs.cribl.io/stream/sources-https/
- Persistent Queues: https://docs.cribl.io/stream/persistent-queues/
- Configuring Persistent Queues: https://docs.cribl.io/stream/4.4/persistent-queues-configuring/

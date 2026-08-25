# Cribl Stream Workshop: Inbound Sources & Outbound Destinations

## Workshop Overview

This hands-on workshop walks through configuring Cribl Stream to receive data via **Syslog** (inbound source) and forward it to **Splunk** (outbound destination). Cribl Stream is already deployed via Docker Compose, and **Persistent Queues (PQ) are pre-configured** — no additional setup for PQ is required in this session.

### Prerequisites

- Cribl Stream instance up and running (Docker Compose deployment)
- Access to the Cribl Stream UI (default: `http://<host>:9000`)
- A Splunk instance (or Splunk HEC endpoint) reachable from the Cribl worker
- A syslog-capable data source or test tool (e.g., `logger`, `netcat`, or a syslog generator)
- Basic familiarity with the Cribl Stream UI (Worker Groups, QuickConnect / Routes, Data tab)

### Learning Objectives

By the end of this workshop, participants will be able to:

1. Configure a Syslog source to ingest data over UDP/TCP into Cribl Stream
2. Verify data is landing correctly using Cribl's live data capture tools
3. Configure a Splunk destination (via Splunk HEC or Splunk TCP)
4. Route data from the Syslog source to the Splunk destination
5. Validate end-to-end data flow from source to destination

### Agenda

| Module | Topic | Duration |
|--------|-------|----------|
| 1 | Introduction & Environment Check | 10 min |
| 2 | Configuring Inbound Sources — Syslog | 30 min |
| 3 | Configuring Outbound Destinations — Splunk | 30 min |
| 4 | Routing & End-to-End Validation | 20 min |
| 5 | Troubleshooting & Q&A | 15 min |

---

## Module 1: Introduction & Environment Check

### 1.1 Confirm Cribl Stream is Running

```bash
docker ps
```

Confirm the Cribl Stream container(s) show a status of `Up`.

### 1.2 Log into the Cribl Stream UI

1. Open a browser and navigate to `http://<host>:9000`
2. Log in with your admin credentials
3. From the top-left **Products** menu, expand **Stream** and select **Worker Groups**
4. Click into your target Worker Group (e.g. `default`) — all Source, Destination, and Routing configuration in this workshop happens *inside* a Worker Group, not at the top level
5. Confirm you can see the Worker Group's navigation (Data > Sources, Data > Destinations, Data > Routing / QuickConnect)

> **Navigation note:** The **Products** menu also lists **Edge** (Fleets / Edge Nodes), which is a separate, lightweight agent product — not used in this workshop. Make sure you're under **Stream > Worker Groups**, not **Edge**.

> **Note:** Persistent Queues are already configured on this instance. You do not need to enable or configure PQ during this workshop — it will already be available as an option on relevant Sources/Destinations if you choose to inspect it.

---

## Module 2: Configuring Inbound Sources — Syslog

### 2.1 Step 1: Configure the Rocky Linux Syslog Client

Before creating the Cribl source, set up the sending side — a Rocky Linux host that will forward its logs to Cribl Stream via `rsyslog`.

**a. Confirm rsyslog is installed and running**

```bash
rpm -q rsyslog
sudo systemctl status rsyslog
```

If not installed:

```bash
sudo dnf install -y rsyslog
sudo systemctl enable --now rsyslog
```

**b. Create a forwarding rule**

```bash
sudo tee /etc/rsyslog.d/90-cribl-forward.conf <<'EOF'
# Forward all logs to Cribl Stream over UDP
*.* @<cribl_host>:5514

# For TCP instead, use double @@ and comment out the UDP line above
# *.* @@<cribl_host>:5514
EOF
```

Replace `<cribl_host>` with your Cribl Stream host's IP or hostname.

**c. Restart rsyslog**

```bash
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

**d. Open the outbound firewall (firewalld, default on Rocky)**

```bash
# UDP
sudo firewall-cmd --permanent --add-port=5514/udp

# TCP (if using TCP instead)
sudo firewall-cmd --permanent --add-port=5514/tcp

sudo firewall-cmd --reload
```

> **Note:** This opens the port on the Rocky Linux client. The rule that matters most is on the **Cribl Stream host**, which needs the inbound port open to actually receive the traffic.

**e. Check SELinux (Rocky enforces it by default)**

Outbound syslog traffic is rarely blocked by SELinux, but if you hit issues:

```bash
sudo ausearch -m avc -ts recent | grep syslog
getsebool -a | grep syslog
```

**f. (Optional) Scope forwarding to specific facilities instead of everything**

```bash
# Only auth and authpriv logs
auth,authpriv.* @<cribl_host>:5514
```

Keep this client configured — you'll use it to send test data once the Cribl-side Syslog source is created in the next step.

### 2.2 Overview

Cribl Stream can receive Syslog data over both UDP and TCP, and supports RFC3164 and RFC5424 formats. In this module, you'll create a Syslog source to listen for incoming log data.

### 2.3 Create a Syslog Source

1. From **Products > Stream > Worker Groups**, click into your Worker Group (e.g. `default`)
2. In the Worker Group's left navigation, go to **Data > Sources**
3. Click **Syslog** under the "Data Sources" list (or search for "Syslog" in the source picker)
4. Click **+ New Source** (or **Add Source**)
5. Configure the following fields:

| Field | Value | Notes |
|-------|-------|-------|
| Input ID | `syslog_in` | A unique, descriptive identifier |
| Address | `0.0.0.0` | Listen on all interfaces |
| UDP Port | `5514` | Avoid privileged port 514 unless running as root |
| TCP Port | `5514` | Optional — enable if TCP delivery is needed |
| Enable | ✅ | Toggle on |

![New Syslog Source — General Settings screen in Cribl Stream, showing Input ID, Description, Address, UDP port, and TCP port fields](assets/syslog_new_source.png)

> The **Input ID** field auto-generates an internal identifier of the form `syslog:<your-id>` (shown as the hint `__inputId.startsWith('syslog::')` below the field) — you only need to type the friendly name, e.g. `syslog_in`.

5. Under **Connection Settings**, leave defaults unless your environment requires custom timeouts
6. Under **Processing Settings**, note the default fields Cribl will auto-populate (`host`, `cribl_pipe`, `source`, etc.)
7. Click **Save**

### 2.4 Verify the Source is Listening

> **Note:** `docker exec ... netstat` often fails because the Cribl container image does not ship `net-tools`/`netstat`. Use one of the alternatives below instead.

**Option A — Check from the Docker host (recommended):**

Since Docker Compose publishes the port to the host, you can check directly from the host machine without entering the container:

```bash
# Linux host
ss -lntu | grep 5514

# macOS host
lsof -i :5514
```

**Option B — Check inside the container with `ss` (usually present even when `netstat` isn't):**

```bash
docker exec -it <cribl_container_name> ss -lntu | grep 5514
```

**Option C — Confirm via the Cribl UI (most reliable):**

1. Go to **Data > Sources > Syslog**
2. Find `syslog_in` in the list
3. Confirm the status indicator is green/healthy — Cribl reports the listener status directly, so this is the most reliable check regardless of what shell tools are available in the container

If the port doesn't appear bound or the source shows unhealthy, double-check that the port is exposed in your `docker-compose.yml` (`ports:` section) and isn't already in use by another process.

### 2.5 Send Test Syslog Data

From a terminal on a machine that can reach the Cribl host — this can be the Rocky Linux client configured in step 2.1 (send a real log event, e.g. `logger "test"`, and it will forward automatically), or any ad-hoc host using the tools below:

```bash
# Using logger (UDP) — also works on the Rocky Linux client itself
logger -n <cribl_host> -P 5514 -d "Test syslog message from workshop"

# Or using netcat (TCP)
echo "<14>Aug 25 12:00:00 host1 testapp: Hello from workshop" | nc <cribl_host> 5514
```

### 2.6 Validate with Live Data Capture

1. In the Cribl UI, go to your `syslog_in` source
2. Click **Live Data** (the capture/preview icon)
3. Click **Start Capture**
4. Send another test message from your terminal
5. Confirm the event appears in the live capture pane with expected fields (`_raw`, `host`, `source`, `sourcetype`)
6. Click **Stop Capture**

**Checkpoint:** ✅ Syslog source is created, listening, and successfully receiving test events.

---

## Module 3: Configuring Outbound Destinations — Splunk

### 3.1 Overview

Cribl Stream supports multiple ways to send data to Splunk:

- **Splunk HTTP Event Collector (HEC)** — recommended, simplest to configure
- **Splunk TCP (Splunk-to-Splunk / S2S)** — for forwarding as if from a Splunk forwarder

This module uses **Splunk HEC** as the primary method.

### 3.2 Prerequisites on the Splunk Side

Before configuring the destination in Cribl, confirm on Splunk:

1. HEC is enabled: **Settings > Data Inputs > HTTP Event Collector**
2. A HEC token has been created and is enabled
3. Note the HEC endpoint (e.g., `https://<splunk_host>:8088`) and the token value
4. Confirm the target index exists and the token is scoped to allow it

### 3.3 Create a Splunk HEC Destination

1. Within the same Worker Group (**Products > Stream > Worker Groups > default**), go to **Data > Destinations**
2. Select **Splunk HEC** from the destination types
3. Click **+ New Destination**
4. Configure the following:

| Field | Value | Notes |
|-------|-------|-------|
| Output ID | `splunk_hec_out` | Unique, descriptive identifier |
| Splunk HEC URLs | `https://<splunk_host>:8088` | Include full URL with port |
| Authentication Token | `<your-hec-token>` | Store as a secret if available |
| Default Index | `main` (or your target index) | |
| Default Source Type | `cribl_workshop` | Optional, helps identify test data |
| TLS Settings | Enable/verify certs as per your environment | Disable cert validation only in lab/test environments |

5. Under **Persistent Queue**, leave the existing configuration as-is (already set up — no changes needed here)
6. Click **Test Connection** to validate connectivity/auth
7. Click **Save**

### 3.4 Verify Destination Health

1. On the **Destinations** list, confirm `splunk_hec_out` shows a green/healthy status indicator
2. If red/unhealthy, check the **Splunk HEC URLs**, token, and network reachability (see Troubleshooting section)

**Checkpoint:** ✅ Splunk HEC destination is created and reporting healthy connectivity.

---

## Module 4: Routing & End-to-End Validation

### 4.1 Connect Source to Destination

**Option A — QuickConnect (simplest, no pipeline logic):**

QuickConnect is configured from *within the source itself*, not a separate top-level page:

1. Go to **Data > Sources > Syslog**, and click into `syslog_in`
2. In the left-hand tab list, click **Connected Destinations**
3. At the top, you'll see a toggle between **Send to Routes** and **QuickConnect** — select **QuickConnect**
4. Under **Use QuickConnect**, click **Add Quick Connection** (a row will appear with **Pipeline or Pack** and **Destination** columns)
5. Leave **Pipeline or Pack** set to blank/`Select one` for a passthrough (no transformation)
6. In the **Destination** dropdown, select `splunk_hec_out` (replace the default `default:default` entry if one is pre-filled)
7. Click **Save**

![Connected Destinations tab on the syslog_in source, showing the QuickConnect toggle and the Pipeline or Pack / Destination row used to link to splunk_hec_out](assets/quickconnect_connected_destinations.png)

> **Tip:** You can add multiple Quick Connection rows to fan the same source out to several destinations.

**Option B — Routes (more control, allows pipelines/filters):**

1. Within your Worker Group, go to **Data > Routing**
2. Click **+ Add Route**
3. Set:
   - **Name:** `syslog_to_splunk`
   - **Filter:** `true` (accept all events for this workshop) or a field-based filter (e.g., `source=='syslog_in'`)
   - **Pipeline:** `(none)` or a passthrough pipeline
   - **Output:** `splunk_hec_out`
4. Click **Save**, then **Commit & Deploy**

### 4.2 Send End-to-End Test Data

```bash
logger -n <cribl_host> -P 5514 -d "End-to-end workshop validation event"
```

### 4.3 Validate in Splunk

1. Log into Splunk Search & Reporting
2. Run a search:

```spl
index=main sourcetype=cribl_workshop
| head 10
```

3. Confirm the test event appears with the expected `_raw` content and timestamp

### 4.4 Validate in Cribl (Monitoring)

1. In Cribl, go to **Monitoring**
2. Review the **Sources** and **Destinations** dashboards
3. Confirm event/byte counts are incrementing for `syslog_in` and `splunk_hec_out`

**Checkpoint:** ✅ Data flows end-to-end from Syslog source through Cribl Stream into Splunk.

---

## Module 5: Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| No data in Live Capture | Wrong port, firewall blocking, source disabled | Verify port bindings, check `docker ps` port mappings, confirm source is enabled |
| Syslog port fails to bind | Port already in use or requires root (<1024) | Use a port ≥1024 or adjust container permissions |
| Splunk destination shows unhealthy | Wrong HEC URL, invalid/disabled token, TLS mismatch | Re-check HEC settings in Splunk, test with `curl` against the HEC endpoint |
| Events reach Cribl but not Splunk | Route/QuickConnect not deployed, filter excluding events | Confirm **Commit & Deploy**, review route filter logic |
| Events in Splunk missing fields | Sourcetype/index misconfigured on destination | Adjust default source type/index on the Splunk HEC destination |

### Quick HEC Connectivity Test (from Cribl host)

```bash
curl -k https://<splunk_host>:8088/services/collector/health \
  -H "Authorization: Splunk <your-hec-token>"
```

Expected response: `{"text":"HEC is healthy","code":17}`

---

## Workshop Wrap-Up

### Summary of What Was Configured

- ✅ Syslog inbound source (`syslog_in`) listening on UDP/TCP 5514
- ✅ Splunk HEC outbound destination (`splunk_hec_out`)
- ✅ Route/QuickConnect linking source to destination
- ✅ End-to-end validation confirmed in Splunk

### Suggested Next Steps (Beyond This Workshop)

- Add a processing Pipeline between source and destination (masking, filtering, enrichment)
- Explore additional inbound sources (HTTP, TCP JSON, Kafka)
- Explore Splunk TCP (S2S) as an alternative destination method
- Review Persistent Queue metrics for the destination (already configured) under **Monitoring**

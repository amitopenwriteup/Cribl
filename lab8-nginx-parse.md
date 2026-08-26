# Phase 1 Lab: Nginx on Rocky Linux → Cribl (Parser Only) → Splunk Cloud

**Objective:** Prove the end-to-end pipe works with the *minimum* moving parts — install Nginx, collect its logs with Cribl, parse them with a single Parser function, and land raw parsed events in Splunk Cloud. No Eval, Mask, Drop, or Aggregation yet — those come in Phase 2 once this path is confirmed.

**Duration:** ~1–1.5 hours
**Prerequisites:**
- Splunk Cloud trial account with HEC enabled
- Cribl Stream already running as a **Leader/Worker cluster** via Docker Compose
- Rocky Linux host running Nginx
- Network connectivity: Nginx host → Cribl → Splunk Cloud

---

## Architecture (Phase 1 scope)

```
┌─────────────────┐
│  Nginx on        │
│  Rocky Linux      │
│  (access.log)     │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────────┐
│  Cribl Stream (Leader/Worker) │
│  ┌──────────────────────────┐│
│  │ cribl-leader   :9000/4200││
│  │ cribl-worker-1 :9001     ││
│  │ cribl-worker-2 :9002     ││
│  └──────────────────────────┘│
│  Collector: nginx_access      │
│        ↓                      │
│  Pipeline: nginx_processing    │
│        [PARSER only]          │
│        ↓                      │
│  Destination: splunk_cloud_hec │
└────────┬──────────────────────┘
         │ HTTPS :8088 (HEC)
         ▼
┌──────────────────────────────┐
│  Splunk Cloud                 │
│  Index: nginx_logs            │
│  Sourcetype: nginx_access      │
└──────────────────────────────┘
```

Everything else from the full lab (Mask, Drop, Aggregation, dashboards, alerts, GeoIP, sampling) is deferred to Phase 2+.

---

## Part 1 — Install and Configure Nginx on Rocky Linux

### Step 1.1 — Install Nginx
```bash
sudo dnf install -y nginx
nginx -v
```

### Step 1.2 — Start and Enable Nginx
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```
Confirm status shows `active (running)`.

### Step 1.3 — Open the Firewall Port
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

### Step 1.4 — Verify Nginx is Running
```bash
curl http://localhost/
```
Expect the Nginx welcome page (HTML, 200 status).

### Step 1.5 — Check Log Locations
```bash
ls -l /var/log/nginx/
```
Expect `access.log` and `error.log`.

### Step 1.6 — Generate Test Traffic
```bash
curl http://localhost/ -s > /dev/null
curl http://localhost/index.html -s > /dev/null
curl http://localhost/missing-page -s > /dev/null
curl http://localhost/forbidden -s > /dev/null
```

### Step 1.7 — Inspect Raw Log Format
```bash
sudo tail -3 /var/log/nginx/access.log
```
Example (combined format):
```
127.0.0.1 - - [19/Aug/2026:12:34:56 +0000] "GET / HTTP/1.1" 200 124 "-" "Mozilla/5.0 (X11; Linux x86_64)"
127.0.0.1 - - [19/Aug/2026:12:34:57 +0000] "GET /missing-page HTTP/1.1" 404 212 "-" "curl/7.68.0"
```

*(error.log is not used in Phase 1 — skip its collector for now to keep the test surface small.)*

---

## Part 2 — Confirm Cribl Cluster and Mount Nginx Logs

### Step 2.1 — Confirm the actual container topology

Your deployment is a **Leader/Worker cluster**, not a single `cribl` container:

```bash
docker ps
```
Expected:
```
CONTAINER ID   IMAGE               NAMES            PORTS
b4d1f688ecef   cribl/cribl:latest  cribl-worker-1   0.0.0.0:9001->9000/tcp
8241b6bb585a   cribl/cribl:latest  cribl-worker-2   0.0.0.0:9002->9000/tcp
9d5e55d55bc0   cribl/cribl:latest  cribl-leader     0.0.0.0:4200->4200/tcp, 0.0.0.0:9000->9000/tcp
```

> **This matters for every command below.** There is no service literally named `cribl` — always target `cribl-leader`, `cribl-worker-1`, or `cribl-worker-2` explicitly. All configuration (Sources, Pipelines, Destinations) happens through the **Leader UI** at `http://localhost:9000`; workers execute the config the leader pushes to them.

### Step 2.2 — Mount the Nginx log directory into the worker containers

The File System Collector runs on workers, so the bind mount needs to reach the workers that will actually read the file. In `docker-compose.yml`:

```yaml
services:
  cribl-worker-1:
    image: cribl/cribl:latest
    container_name: cribl-worker-1
    volumes:
      - /var/log/nginx:/var/log/nginx:ro
    # ... existing config

  cribl-worker-2:
    image: cribl/cribl:latest
    container_name: cribl-worker-2
    volumes:
      - /var/log/nginx:/var/log/nginx:ro
    # ... existing config
```

Apply the change (recreates only the affected containers):
```bash
docker compose up -d
docker compose ps
```

### Step 2.3 — Verify the mount is visible inside the workers

```bash
docker compose exec cribl-worker-1 ls -l /var/log/nginx/
docker compose exec cribl-worker-2 ls -l /var/log/nginx/
```
Both should list `access.log`.

> **Permissions note:** Nginx logs are typically `root`-owned, mode `640`. If the container's runtime user can't read them, you'll see a permission error here. Lab-only quick fix:
> ```bash
> sudo chmod 644 /var/log/nginx/access.log
> ```

### Step 2.4 — Confirm the Leader UI and API are reachable

```bash
curl http://localhost:9000/ -I
```
Expect HTTP 200.

Metrics check (correct container name — this is what failed before):
```bash
docker compose exec cribl-leader curl http://localhost:9000/api/v1/system/metrics \
  -H "Authorization: Bearer <your-cribl-token>" | jq .
```

---

## Part 3 — Create Cribl Data Source (Nginx Access Log Only)

### Step 3.1 — Access Cribl Leader UI
Go to `http://localhost:9000` and log in.

### Step 3.2 — Navigate to Collectors
Left sidebar → **Data → Sources → Collectors** → **+ Add Collector**

### Step 3.3 — Create a File System Collector

**Collector Type:** **File System**

**Collector Settings tab:**
```
Collector ID:       nginx_access
Description:        Collect Nginx access logs (Phase 1)
Directory*:         /var/log/nginx/access.log
```

**Optional Settings:**
```
Recursive:           ON (default; harmless on a single file)
Batch size limit:    10
Destructive:         OFF
Encoding:            utf8
```

**Result Settings tab:**
- **Fields** → add static field: `sourcetype = nginx_access`
- **Result Routing** → set destination **Pipeline** to `nginx_processing` (created in Part 4)

**Saving:**
- Click **Save & Run** to test immediately with a one-time collection
- Click **Save & Schedule** with a cron like `* * * * *` once you're ready for it to run continuously

### Step 3.4 — Verify the Collector

**Sources → Collectors** → confirm `nginx_access` shows:
- **Status:** Enabled
- **Latest Ad Hoc Run:** a timestamp with non-zero event count once triggered

---

## Part 4 — Create the Phase 1 Pipeline (Parser Only)

### Step 4.1 — Navigate to Pipelines
Left sidebar → **Processing → Pipelines** → **+ Add Pipeline**

### Step 4.2 — Create the Pipeline
```
Pipeline ID:    nginx_processing
Description:    Phase 1 — parser only, prove connectivity
```
Click **Create**.

### Step 4.3 — Confirm the Collector Feeds This Pipeline
Already linked via Step 3.3's Result Routing. Confirm from either:
- **Sources → Collectors → nginx_access → Result Settings → Result Routing**, or
- **Pipelines → nginx_processing** preview, after a run has occurred

### Step 4.4 — Add the ONLY function for Phase 1: PARSER

Click **+ Add Function** → **Select: Parser**

```
Parser Type:      Regex
Source Field:     _raw

Regex Pattern:
(?<remote_ip>[\d.]+)\s-\s(?<remote_user>\S+)\s\[(?<timestamp>[^\]]+)\]\s"(?<method>\S+)\s(?<uri>\S+)\s(?<protocol>[^"]+)"\s(?<status>\d+)\s(?<bytes>\d+)
```

Click **Save**.

### Step 4.5 — Do NOT add Eval, Mask, Drop, or Aggregation yet

```
nginx_access (Collector)
      ↓
  [PARSER]     ← only function
      ↓
  [DESTINATION]
```

These are deliberately deferred to Phase 2 so that if something breaks, you know it's not a downstream function causing it.

### Step 4.6 — Enable the Pipeline
Toggle **Pipeline Enabled** → **ON**. Click **Save Pipeline**.

---

## Part 5 — Configure and Test the Splunk Cloud Destination

### Step 5.0 — Find the correct HEC hostname first

Do **not** guess the `http-inputs-<stack>.splunkcloud.com` pattern — confirm it directly:

1. Splunk Cloud → **Settings → Data Inputs → HTTP Event Collector**
2. Open your token (or create one) and note the **HEC endpoint** shown
3. Depending on your Splunk Cloud experience (Classic vs Victoria), the pattern may be:
   - `http-inputs-<stack>.splunkcloud.com:8088` (Classic)
   - `inputs.<stack>.splunkcloud.com:8088` or `<stack>.splunkcloud.com:8088` (Victoria)
4. Verify DNS resolves:
```bash
nslookup <hostname-shown-in-splunk-ui>
```
Must return a real IP, not `NXDOMAIN`, before continuing.

### Step 5.1 — Create the Destination

**Leader UI → Data → Destinations → + Add Destination**

**Destination Type:** **Splunk** → **HTTP Event Collector**

```
Destination ID:        splunk_cloud_hec
Description:           Send to Splunk Cloud via HEC (Phase 1 test)

Host:                  <hostname confirmed in Step 5.0>
Port:                  8088
Token:                 <HEC token from Splunk Cloud>

SSL Certificate:       Verify (checked)
Index:                 nginx_logs
Sourcetype:            nginx_access

Max Content Length:    2097152
Timeout:               30s
Connection Timeout:    10s
```
Click **Save**. This config is pushed automatically to both `cribl-worker-1` and `cribl-worker-2`.

### Step 5.2 — Test the Destination

**Data → Destinations → splunk_cloud_hec → Test**

Expected: **"Connection successful"**

**If it fails, check in this order:**

**a) DNS**
```bash
nslookup <hostname>
```

**b) Outbound reachability from a worker (workers send the data, not the leader)**
```bash
docker compose exec cribl-worker-1 curl -v https://<hostname>:8088/services/collector/health \
  -H "Authorization: Splunk <your-hec-token>"
```
Expect HTTP 200 with a JSON health response.

**c) Token validity**
- Splunk Cloud → **Settings → HTTP Event Collector** → token is **Enabled**
- Token's allowed indexes include `nginx_logs`

**d) Firewall/security group**
- Outbound port `8088` allowed from the Docker host

Do not proceed to Part 6 until this test passes.

---

## Part 6 — Wire Pipeline Output to the Destination

### Step 6.1 — Open the Pipeline
**Processing → Pipelines → nginx_processing**

### Step 6.2 — Connect Output
After the **PARSER** function, add a **Destination** node → select `splunk_cloud_hec`.

### Step 6.3 — Confirm Routing
```
Index:       nginx_logs
Sourcetype:  nginx_access
```
Click **Save**.

### Step 6.4 — Confirm Pipeline is Enabled
Toggle at top of editor should read **ON**. Click **Save Pipeline**.

---

## Part 7 — Validate Phase 1 End-to-End

### Step 7.1 — Generate Fresh Traffic
```bash
for i in {1..10}; do
  curl http://localhost/ -s > /dev/null
  curl http://localhost/test$i -s > /dev/null
done
```

### Step 7.2 — Trigger the Collector
**Sources → Collectors → nginx_access → Actions → Run** (or **Save & Run** from the editor).

### Step 7.3 — Check Cribl-Side Metrics
**Leader UI → Monitoring** (or **Settings → System Status**):
- `nginx_access` collector: non-zero **Latest Ad Hoc Run** event count
- `nginx_processing` pipeline: events flowing through the Parser stage
- `splunk_cloud_hec` destination: events sent, no error count climbing

### Step 7.4 — Search in Splunk Cloud

**Search & Reporting**, time range **Last 15 minutes**:
```spl
index=nginx_logs sourcetype=nginx_access
```

You should see events with fields extracted by the Parser:
- `remote_ip`
- `remote_user`
- `timestamp`
- `method`
- `uri`
- `protocol`
- `status`
- `bytes`

If fields are missing but `_raw` looks intact, recheck the Parser regex against your actual log format (Nginx's `log_format` directive may differ from the default combined format).

---

## Part 8 — Phase 1 Troubleshooting Reference

| Symptom | Cause | Fix |
|---|---|---|
| `docker compose exec cribl ...` → "service cribl is not running" | Wrong container name — this is a leader/worker cluster, not a single `cribl` service | Use `cribl-leader`, `cribl-worker-1`, or `cribl-worker-2` explicitly |
| `nslookup http-inputs-<stack>.splunkcloud.com` → NXDOMAIN | Guessed hostname pattern is wrong for your stack | Get the exact HEC hostname from Splunk Cloud → Settings → HTTP Event Collector |
| Collector shows 0 events | Worker container can't see the file, wrong path, or run hasn't fired | `docker compose exec cribl-worker-1 ls -l /var/log/nginx/`; verify Directory field matches the container path; trigger **Save & Run** |
| Permission denied reading the log file | Root-owned log, container user can't read it | `sudo chmod 644 /var/log/nginx/access.log` (lab only) |
| Destination Test fails after correct hostname | Token disabled/wrong, or firewall blocking 8088 outbound | Check token status in Splunk Cloud; test raw curl from `cribl-worker-1`; confirm security group/firewall allows 8088 outbound |
| Events reach Splunk but no parsed fields | Parser regex doesn't match actual log format | `tail` the raw log, compare against the regex, adjust for custom Nginx `log_format` |

---

## Phase 1 Completion Checklist

- [ ] Nginx installed, running, generating `access.log` on Rocky Linux
- [ ] `docker ps` confirms `cribl-leader`, `cribl-worker-1`, `cribl-worker-2` all `Up`
- [ ] `/var/log/nginx` bind-mounted **read-only** into both worker containers and verified readable
- [ ] `nginx_access` File System Collector created, routed to `nginx_processing`
- [ ] Pipeline `nginx_processing` contains **Parser only** — no Eval/Mask/Drop/Aggregation
- [ ] Correct HEC hostname obtained from Splunk Cloud UI (not guessed) and confirmed via `nslookup`
- [ ] `splunk_cloud_hec` destination created and **Test** shows "Connection successful"
- [ ] Pipeline wired to destination and enabled
- [ ] Fresh traffic generated, collector run triggered
- [ ] Events visible in `index=nginx_logs sourcetype=nginx_access` in Splunk Cloud with parsed fields present

---

## Next Steps (Phase 2, once Phase 1 is fully green)

1. Add **Eval** function for calculated fields (`is_error`, `bytes_mb`, `request_summary`)
2. Add **Mask** function for PII redaction of `remote_ip` / `remote_user`
3. Add **Drop** function to filter out 200–299 status events
4. Add **Aggregation** (optional) for 1-minute rollups
5. Add the `nginx_error` collector and its own simpler pipeline
6. Build Splunk Cloud dashboards and alerts

Add and verify these one at a time in Splunk after each addition — don't stack all four before testing again.

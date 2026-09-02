# Hands-On Lab: OTel Collector → Cribl Stream → Backend

A self-contained lab you can run on your laptop with Docker. You'll stand up an OTel Collector, generate real traces/metrics/logs, ingest them into Cribl Stream over OTLP, build a small OTel-aware pipeline, and route the result to a destination you can inspect.

**Time:** ~45–60 minutes
**Requires:** Docker + Docker Compose, a free-tier or trial Cribl.Cloud org (or a local Cribl Stream container), a terminal

---

## 0. What you'll build

```
telemetrygen (fake app) ──OTLP──▶ OTel Collector ──OTLP──▶ Cribl Stream ──▶ File / Live Data
                                    (batches,                (OTel Source →
                                     adds attrs)              Pipeline → Destination)
```

By the end, you'll be able to see individual spans, metric data points, and log records land in Cribl, get reshaped by a pipeline, and flow out to a destination.

---

## 1. Get a Cribl Stream instance

Pick **one**:

**Option A — Cribl.Cloud (easiest, no local install)**
1. Sign up for a free trial at `cribl.io` if you don't have an org already.
2. Go to **Products → Stream**, and note your default Worker Group (usually `default`).
3. Under **Manage → Workspace → Data Sources**, note the ports Cribl.Cloud has opened for you — on managed Worker Groups, port `4318` (OTLP/HTTP) isn't available, so you'll be assigned a port in the `20000–20010` range for HTTP, or you can use gRPC on `4317`.

**Option B — Local Cribl Stream via Docker**
```bash
docker run -d --name cribl \
  -p 9000:9000 \
  -p 4317:4317 \
  -p 4318:4318 \
  cribl/cribl:latest
```
Log in at `http://localhost:9000` (default creds are printed in `docker logs cribl` on first boot — change the password immediately).

---

## 2. Create the OTel Source in Cribl

1. In Cribl Stream, go to **Data → Sources** (or **QuickConnect**) and click **Add Source → OpenTelemetry (OTel)**.
2. Configure:
   | Field | Value |
   |---|---|
   | Input Id | `lab-otel-in` |
   | Protocol | `gRPC` |
   | Port | `4317` (local) or your assigned Cloud port |
   | Extract spans | **On** — so each span becomes its own event, easier to inspect |
   | Extract metrics | **On** |
   | Extract logs | **On** |
   | Authentication | `None` for the lab (use Auth Token in anything beyond a lab) |
3. Save, then **Enable** the Source.
4. Open the Source's **Live Data** tab and leave it open in a browser tab — you'll watch events land here in a few minutes.

---

## 3. Run an OTel Collector that forwards to Cribl

Create a working folder and drop in this Collector config. It receives OTLP locally and forwards straight to your Cribl OTel Source.

**`otel-collector-config.yaml`**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 2s
  resource:
    attributes:
      - key: lab.name
        value: "otel-cribl-lab"
        action: insert

exporters:
  otlp/cribl:
    endpoint: "<CRIBL_HOST>:<CRIBL_PORT>"   # e.g. localhost:4317, or your Cribl.Cloud worker endpoint
    tls:
      insecure: true                          # lab only — use real TLS outside a lab

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [otlp/cribl]
    metrics:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [otlp/cribl]
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [otlp/cribl]
```

Replace `<CRIBL_HOST>:<CRIBL_PORT>` — if Cribl is running in the same Docker network use the container name; if it's on your host use `host.docker.internal:4317` (Mac/Windows) or your machine's IP (Linux).

Run the Collector:
```bash
docker run -d --name otelcol \
  -p 4317:4317 -p 4318:4318 \
  -v "$PWD/otel-collector-config.yaml":/etc/otelcol/config.yaml \
  otel/opentelemetry-collector-contrib:latest \
  --config /etc/otelcol/config.yaml
```

> If Cribl is also running locally, you'll have a port clash on 4317. Either run the Collector's receiver on a different port (e.g. `4327`) and point your test app there, or run Cribl's OTel Source on a different port than the Collector's exporter target — just keep the two straight.

---

## 4. Generate real telemetry

Use the OpenTelemetry `telemetrygen` tool to fire synthetic traces, metrics, and logs at your Collector without writing any app code.

```bash
docker run --rm otel/opentelemetry-collector-contrib:latest telemetrygen traces \
  --otlp-endpoint host.docker.internal:4317 --otlp-insecure \
  --duration 30s --rate 5

docker run --rm otel/opentelemetry-collector-contrib:latest telemetrygen metrics \
  --otlp-endpoint host.docker.internal:4317 --otlp-insecure \
  --duration 30s --rate 5

docker run --rm otel/opentelemetry-collector-contrib:latest telemetrygen logs \
  --otlp-endpoint host.docker.internal:4317 --otlp-insecure \
  --duration 30s --rate 5
```

**Checkpoint:** flip back to the Cribl **Live Data** tab from Step 2. You should see a stream of individual span, metric, and log events within a few seconds. If you see nothing:
- Confirm the Collector logs show successful exports (`docker logs otelcol`) — a connection refused means the endpoint/port is wrong.
- Confirm the Cribl Source is **enabled**, not just saved.
- Check that no firewall/security group is blocking the port if Cribl is remote.

---

## 5. Build an OTel-aware Pipeline

Now shape the data before it goes anywhere else.

1. In Cribl, go to **Processing → Pipelines → Add Pipeline**, name it `otel-lab-pipeline`.
2. Add these Functions in order:

   **a. Eval** — tag every event with where it came from
   ```
   Add field: pipeline_stage = "lab-processed"
   ```

   **b. Drop noisy spans** (traces only) — filter condition:
   ```
   __inputId=='lab-otel-in' && name=='healthcheck'
   ```
   Set this Function's filter to that expression and choose **Drop Event** — this simulates dropping high-volume, low-value spans like health checks.

   **c. Mask/redact an attribute** (simulate PII redaction) — Mask Function:
   ```
   Fields: attributes.http.url
   Pattern: (?<=user=)[^&]+
   Replace: ****
   ```

   **d. Sampling** (traces only) — add a **Sampling** Function set to keep, e.g., 50% of remaining trace events, to show volume reduction.

3. Save the pipeline.

4. Go back to your OTel Source's **Connected Pipelines / Routes**, and route events matching `__inputId=='lab-otel-in'` to `otel-lab-pipeline`.

**Checkpoint:** In **Live Data**, confirm events now carry `pipeline_stage: "lab-processed"`, that `healthcheck` spans no longer appear, and that the masked field shows `****` instead of the original value.

---

## 6. Send it somewhere you can verify

For the lab, the simplest destination is the filesystem — it needs no external account and you can `cat` the result.

1. Go to **Data → Destinations → Add Destination → Filesystem**.
2. Set:
   | Field | Value |
   |---|---|
   | Output Id | `lab-file-out` |
   | Destination path | `/tmp/cribl-otel-lab` |
   | Format | `JSON` |
3. Save and enable it.
4. Back in **Routes** (or QuickConnect), connect `otel-lab-pipeline`'s output to `lab-file-out`.
5. Re-run the `telemetrygen` commands from Step 4.
6. Check the output:
   ```bash
   docker exec cribl ls /tmp/cribl-otel-lab
   docker exec cribl sh -c "cat /tmp/cribl-otel-lab/*/*.json | head -5"
   ```

You should see processed JSON events — each one a span, metric data point, or log record, already carrying your pipeline's changes.

---

## 7. Optional extensions

Once the basic flow works, try any of these to go deeper:

- **Fan out to two destinations** — add a second Destination (e.g., an OTLP Destination pointed at a free Grafana Cloud or Honeycomb trial) and route the *same* pipeline output to both the file and the real backend, to see one query's worth of data land in two places.
- **Split traces vs. logs vs. metrics into separate pipelines** — duplicate the pipeline, filter one copy to `type=='span'`, one to `type=='metric'`, one to `type=='log'` (the extracted-event `type` field differs per signal), and give each its own processing logic — mirroring the "OTel-aware pipelines" pattern from the deck.
- **Send matching data to Cribl Lake** — if your org has Lake enabled, add a Lake Destination and route a sampled/raw copy there for long-term storage, while the filtered copy goes to your live backend.
- **Add authentication** — switch the OTel Source's Auth setting to **Auth Token**, update the Collector's `otlp/cribl` exporter with a `headers: {authorization: "Bearer <token>"}` block, and confirm unauthenticated `telemetrygen` calls are now rejected.

---

## Troubleshooting quick-reference

| Symptom | Likely cause |
|---|---|
| No events in Live Data at all | Collector can't reach Cribl — check exporter endpoint/port and `docker logs otelcol` |
| Events arrive but pipeline changes don't show | Route isn't pointing at the pipeline, or pipeline isn't saved/committed |
| `connection refused` from telemetrygen | Wrong port, or Collector container not up (`docker ps`) |
| Cribl.Cloud port 4318 doesn't work | Expected — HTTP/4318 isn't exposed on managed Worker Groups; use gRPC/4317 or an assigned 20000–20010 port |
| Masked field still shows raw value | Regex pattern didn't match — test the Mask Function's pattern against a sample event in the pipeline preview |

---

## Clean up

```bash
docker rm -f otelcol cribl 2>/dev/null
rm -f otel-collector-config.yaml
```

# Lab: OTel Collector → Cribl Stream (gRPC)

```
telemetrygen ──gRPC:4317──▶ OTel Collector ──gRPC:4317──▶ Cribl Stream
```

---

## 1. Cribl OTel Source

Create/edit the OTel Source in Cribl with these settings:

| Field | Value |
|---|---|
| Input ID | `otel` |
| Protocol | `gRPC` |
| Address | `0.0.0.0` |
| Port | `4317` |
| Extract spans | On |
| Extract metrics | On |
| Extract logs | On |
| Enabled | On |

Save, then **Commit & Deploy** if this is a distributed Worker Group. Open the **Live Data** tab and click **Capture** — Live Data only shows events that arrive while a capture session is running.

---

## 2. Create the Collector config file

On the host running Docker (`master`), create the collector config:

```bash
cat > /root/otel-collector-config.yaml << 'EOF'
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
  debug:
    verbosity: detailed
  otlp/cribl:
    endpoint: "10.0.5.1:4317"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlp/cribl]
    metrics:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlp/cribl]
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlp/cribl]
EOF
```

Replace `10.0.5.1` with your actual Cribl host IP.

**Verify the file was created correctly, as a real file (not a directory) with real content:**

```bash
ls -la /root/otel-collector-config.yaml
cat /root/otel-collector-config.yaml
```

You should see file permissions/size on the `ls` line (not `d` at the start, which would mean directory), and the full YAML content printed by `cat`.

---

## 3. Run the Collector

Remove any old container first, so a previous broken mount doesn't get reused:

```bash
docker rm -f otelcol
```

Start the Collector, using the **absolute path** to the config file and `--network host` (required — without it, the container's traffic gets NAT'd through Docker's bridge network and can silently fail to reach Cribl even when the host itself can reach it fine):

```bash
docker run -d --name otelcol \
  --network host \
  -v /root/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml \
  otel/opentelemetry-collector-contrib:latest \
  --config /etc/otelcol-contrib/config.yaml
```

Check it started cleanly:

```bash
docker logs otelcol -f
```

Look for `Everything is ready. Begin running and processing data.` with no `connection refused` or `broken pipe` warnings tied to the `otlp/cribl` exporter.

---

## 4. Generate telemetry

```bash
docker run --rm --network host \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
  --otlp-endpoint 127.0.0.1:4317 --otlp-insecure \
  --duration 30s --rate 5
```

Swap `traces` for `metrics` or `logs` to generate the other signal types, same flags otherwise.

**Checkpoint:**
- `docker logs otelcol -f` should show detailed span/metric/log dumps from the `debug` exporter as traffic is sent.
- Cribl's **Live Data** tab (with Capture already running) should show the same events landing within seconds.

---

## 5. Loop it (optional)

```bash
while true; do
  docker run --rm --network host \
    ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
    --otlp-endpoint 127.0.0.1:4317 --otlp-insecure --duration 20s --rate 5
  docker run --rm --network host \
    ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest metrics \
    --otlp-endpoint 127.0.0.1:4317 --otlp-insecure --duration 20s --rate 5
  docker run --rm --network host \
    ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest logs \
    --otlp-endpoint 127.0.0.1:4317 --otlp-insecure --duration 20s --rate 5
  sleep 5
done
```

---

## Quick diagnostic checklist

| Symptom | Cause / Fix |
|---|---|
| `mount ... not a directory` on `docker run` | Stale container/mount from an earlier run where the config file didn't exist yet — `docker rm -f otelcol`, confirm the host file is a real file with `ls -la` / `cat`, then re-run |
| `write: broken pipe` to Cribl IP | Collector running without `--network host` |
| `connection refused` to Cribl IP | Cribl Source down/reinitializing, or Protocol/Port mismatch on the Source |
| Live Data shows nothing despite traffic flowing | Capture wasn't started before sending traffic; Extract toggles off; config saved but not deployed |
| `debug` exporter shows data, but Cribl shows nothing | Network path confirmed fine — check Cribl Source's Status tab (in/out counters) and Commit & Deploy status next |

---

## Clean up

```bash
docker rm -f otelcol
rm -f /root/otel-collector-config.yaml
```

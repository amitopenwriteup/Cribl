# Lab: OTel Collector → Cribl Stream

```
telemetrygen ──gRPC:4317──▶ OTel Collector ──HTTP:4318──▶ Cribl Stream
```

---

## 1. Cribl OTel Source

Create/edit the OTel Source in Cribl with these settings:

| Field | Value |
|---|---|
| Input ID | `otel` |
| Protocol | `HTTP` |
| Address | `0.0.0.0` |
| Port | `4318` |
| Extract spans | On |
| Extract metrics | On |
| Extract logs | On |
| Enabled | On |

Save and open the **Live Data** tab. Click **Capture** to start watching (Live Data only shows events that arrive while capture is active).

---

## 2. Collector config

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
  debug:
    verbosity: detailed
  otlphttp/cribl:
    endpoint: "http://<CRIBL_HOST>:4318"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlphttp/cribl]
    metrics:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlphttp/cribl]
    logs:
      receivers: [otlp]
      processors: [resource, batch]
      exporters: [debug, otlphttp/cribl]
```

Replace `<CRIBL_HOST>` with your Cribl host's IP or hostname.

**`docker-compose.yaml`**
```yaml
services:
  otelcol:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otelcol
    network_mode: host
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml
    command: ["--config=/etc/otelcol-contrib/config.yaml"]
    restart: unless-stopped
```

Run it:
```bash
docker compose up -d
docker compose logs -f otelcol
```

The `debug` exporter prints every span/metric/log the Collector processes, so `docker compose logs -f otelcol` shows data flowing in real time alongside the export to Cribl.

---

## 3. Generate telemetry

```bash
docker run --rm --network host \
  ghcr.io/open-telemetry/opentelemetry-collector-contrib/telemetrygen:latest traces \
  --otlp-endpoint 127.0.0.1:4317 --otlp-insecure \
  --duration 30s --rate 5
```

Swap `traces` for `metrics` or `logs` to generate the other signal types.

**Checkpoint:** watch `docker compose logs -f otelcol` for span/metric/log dumps, and Cribl's **Live Data** (with Capture running) for the same events arriving.

---

## 4. Loop it (optional)

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

## Clean up

```bash
docker compose down
rm -f otel-collector-config.yaml docker-compose.yaml
```

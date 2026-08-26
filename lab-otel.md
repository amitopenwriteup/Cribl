# Lab: Configuring the OpenTelemetry (OTel) Source in Cribl Stream (HTTP Only)

## Overview

In this lab, you will configure a Cribl Stream Source that receives trace, metric, and log data sent using the OpenTelemetry Protocol (OTLP) **over HTTP transport**. By the end, you'll have a working HTTP-based OTel Source, understand its extraction options, and know how to secure it with TLS and authentication.

**Estimated time:** 45–60 minutes

**What you'll learn:**
- How the OTel Source ingests OTLP data over HTTP (binary Protobuf)
- How to configure General Settings for HTTP, including span/metric/log extraction
- How to set up Authentication (including multiple auth tokens, HTTP-only) and TLS
- How to configure Persistent Queue (PQ) settings
- How to configure HTTP-specific Advanced Settings (health check endpoint, request limits, allow/deny lists)
- How to troubleshoot a live OTel Source

> **Why HTTP?** The OTel Source supports both gRPC and HTTP transports. This lab uses HTTP throughout because it supports multiple auth tokens per Source, exposes a dedicated health check endpoint, and offers finer-grained request/connection controls — all covered below.

---

## Prerequisites

- Access to a Cribl Stream instance (Cloud or on-prem) with permissions to create Sources
- A Worker Group to configure
- (Optional, for TLS exercise) OpenSSL installed locally, or access to a terminal
- (Optional) An OpenTelemetry Collector, or another OTLP-compliant sender, for end-to-end testing

---

## Part 1: Understand the Data Model

Before configuring anything, review the three signal types this Source supports:

| Signal | Structure |
|---|---|
| **Trace** | A tree of **spans**; each span represents work done by one service/component in a request |
| **Metric** | Contains one or more **data points** (aggregated statistical measurements) |
| **Log** | A recording of a single event |

**Checkpoint question:** If extraction is turned off for all three signal types, how many events does Cribl Stream generate per incoming OTLP payload?

<details>
<summary>Answer</summary>
One event per payload — the Source passes through the entire batch as a single event.
</details>

---

## Part 2: Create the OTel Source

1. Navigate to **Products** > **Cribl Stream**, then select a **Worker Group** under **Worker Groups**.
2. Choose a configuration path:
   - **QuickConnect**: Go to **Routing** > **QuickConnect**, select **Add Source**, then pick **OTel** from the list.
   - **Routes**: Go to **Data** > **Sources**, select **OTel**, then select **Add Source**.
3. In the **New Source** modal, fill in **General Settings**:

   | Field | Value to use in this lab |
   |---|---|
   | Input ID | `otel-lab-http-01` |
   | Description | `Lab OTel source (HTTP) for traces/metrics/logs` |
   | OTLP version | `1.3.1` (default) |
   | Protocol | `HTTP` |
   | Address | `0.0.0.0` (default) |
   | Port | `4318` (you must set this manually — the Source defaults to `4317`, which is the gRPC port) |

4. Leave **Extract spans**, **Extract metrics**, and **Extract logs** toggled **off** for now — you'll test this behavior in Part 3.
5. Add a tag, e.g. `lab`, to help you find this Source later.
6. Select **Save**, then **Commit & Deploy**.

**Checkpoint question:** Why must you manually change the **Port** field when using HTTP, instead of relying on the default?

<details>
<summary>Answer</summary>
The Source's **Port** field defaults to `4317`, which is the OTLP default for gRPC. Because this lab uses HTTP, you must change it to `4318`, the OTLP default for HTTP — otherwise your HTTP sender won't be listening on the expected port.
</details>

> **Note:** Only Binary Protobuf payload encoding is supported over HTTP. JSON Protobuf is **not** supported. Make sure your OTLP/HTTP sender is configured to export binary Protobuf.
>
> On Cribl.Cloud, port `4318` is not available on Cribl-managed Worker Groups — if you're running this lab in Cribl.Cloud, use a hybrid Worker Group or choose a different port and update your sender accordingly.

---

## Part 3: Test Extraction Settings

1. Reopen your `otel-lab-01` Source configuration.
2. Toggle **Extract spans** on. Save and commit.
3. Using an OTLP sender (or a test/curl payload if you have one prepared), send a trace payload containing multiple spans.
4. On the **Live Data** tab, click **Start Capture** and observe the output.

**Checkpoint question:** With **Extract spans** enabled, how many events would you expect to see if you sent a single trace payload containing 5 spans?

<details>
<summary>Answer</summary>
5 individual span events, instead of 1 combined payload event.
</details>

5. Repeat the exercise for **Extract metrics** (generates one event per data point) and, if using OTLP `1.3.1`, **Extract logs** (generates one event per log record).

> **Note:** **Extract logs** is only available when **OTLP version** is set to `1.3.1`.

---

## Part 4: Configure Authentication (HTTP)

1. Open your Source's configuration and go to the **Authentication** section.
2. Select **Auth token**.
3. Enter a bearer token value, e.g. `lab-secret-token-123`.
4. Save and commit.
5. Configure your OTel client/Collector to send the credential as an HTTP request header:

   | Authentication type | Key | Value |
   |---|---|---|
   | Auth token | `authorization` | `Bearer lab-secret-token-123` |

   Use lowercase `authorization` for the header key.

**Checkpoint question:** Why is the credential sent as an HTTP header here, rather than as gRPC metadata?

<details>
<summary>Answer</summary>
Because Protocol is set to `HTTP` in this lab, senders include credentials in HTTP request headers. (Only when Protocol is `gRPC` would the same key/value be sent as gRPC Metadata instead.)
</details>

### Multiple Auth Tokens (HTTP-only feature)

Because this Source uses HTTP, you can configure **multiple auth tokens** on the same Source — this option is not available on gRPC. Use this when several OTLP/HTTP senders (apps, services, tenants) need to share one Source with different credentials.

1. In **Authentication**, add a second token entry for a different sender.
2. Add an **Authentication Field** to this token, e.g. `app=checkout`.
3. Add a third token with a Field such as `app=payments`.
4. Save and commit.

Each Field is a name/value pair added to the resulting events when a client authenticates with that token — useful for routing or filtering events later by sender.

**Checkpoint question:** Two different applications need to send OTLP/HTTP data to the same Cribl OTel Source, and you want to tell their events apart downstream. What's the simplest way to do this on an HTTP Source?

<details>
<summary>Answer</summary>
Configure two separate auth tokens on the Source, each with its own **Authentication Field** (e.g. `app=checkout` and `app=payments`). The Field values get added to each sender's events automatically, letting you route or filter by sender.
</details>

---

## Part 5: Secure the Source with TLS

1. Generate a self-signed certificate and key (lab/test use only):

   ```shell
   openssl req -nodes -new -x509 -newkey rsa:2048 -keyout myKey.pem -out myCert.pem -days 420
   ```

2. In your Source's **TLS Settings (Server Side)** tab, toggle **Enabled** on.
3. Configure:
   - **Certificate**: select or create using `myCert.pem`
   - **Private key path**: path to `myKey.pem`
   - **Certificate path**: path to `myCert.pem`
   - Leave **CA certificate path** empty
   - Leave **Authenticate client (mutual auth)** toggled off
4. Save and commit.
5. On your OTel Collector, update the `exporters.otlphttp` section of your config (e.g. `otel-config.yaml`), pointing at your HTTP port (`4318`):

   ```yaml
   exporters:
     otlphttp:
       endpoint: "https://<Cribl_IP_address>:4318"
       tls:
         insecure: false
         insecure_skip_verify: true
   ```

   > Note the exporter name is `otlphttp` (not `otlp`) when your Collector is sending over HTTP transport.

**Checkpoint question:** Why is `insecure_skip_verify: true` required in this example, even though TLS is enabled?

<details>
<summary>Answer</summary>
Because a self-signed certificate is used, and mutual auth (client certificate validation) is disabled server-side — the client needs to skip strict certificate verification to connect successfully.
</details>

---

## Part 6: Configure Persistent Queue (PQ)

1. Go to the **Persistent Queue Settings** tab.
2. Toggle **Enable persistent queue** on.
3. Set **Mode** to `Always On`.
4. Leave **Buffer size limit (bytes)** at the default `1MB`.
5. Set **Queue size limit** to `5 GB` (default).
6. Set **Compression** to `Gzip`.
7. Save and commit.

**Checkpoint question:** Your downstream Destination goes offline for maintenance. With PQ in `Always On` mode, what happens to incoming events?

<details>
<summary>Answer</summary>
Events are written to disk-based persistent queue first, then forwarded once the Destination processing engine is available again — so no data is lost during the outage.
</details>

> **Note:** On Cribl-managed Cloud Worker Groups (Enterprise plan), PQ configuration is limited to the on/off toggle; it auto-configures in `Always On` mode with a 1 GB per-Source, per-Worker-Process limit.

---

## Part 7: Configure HTTP-Specific Advanced Settings

Because Protocol is set to `HTTP`, your Source exposes additional Advanced Settings not available on gRPC. Configure the following:

1. Open your Source's **Advanced Settings** tab.
2. Toggle on **Health check endpoint**. This exposes `http(s)://<host>:4318/cribl_health`, returning `200` when the Source is healthy.
3. Set **Active request limit (on-prem only)** — leave at the default `256`, or lower it if you want to test backpressure behavior.
4. Set **Requests-per-socket limit** to `100` (default is `0`, unlimited) to force clients to open new connections periodically, spreading load across more Worker Node processes.
5. Leave **Socket timeout (seconds)** and **Request timeout (seconds)** at their defaults (`0` = wait indefinitely) for this lab.
6. Set **Keep-alive timeout (seconds)** to `10` (default `5`) — you'll revisit this in Part 8's troubleshooting scenario.
7. Leave **IP allowlist regex** at `.*` and **IP denylist regex** at `^$` (both defaults = allow all) unless you want to restrict access to a specific test client's IP.
8. Save and commit.

**Checkpoint question:** You verify your Source is healthy by curling `http://<host>:4318/cribl_health` and get a `503` response with the message `Server is busy, max active connections reached`. What setting would you investigate?

<details>
<summary>Answer</summary>
**Active request limit (on-prem only)** — a `503`/"max active connections reached" error indicates too many concurrent connections per Worker Process against that limit.
</details>

**Checkpoint question:** You have a client sending thousands of requests over only a couple of long-lived connections (hours). What should you consider adjusting, and why?

<details>
<summary>Answer</summary>
Lower the **Requests-per-socket limit**. Once the limit is hit, Cribl Stream tells the client (via the `Connection: close` header) to open a new connection, spreading the workload across more Worker Node processes instead of concentrating it on one.
</details>

---

## Part 8: Troubleshooting Practice

1. Go to the **Live Data** tab and click **Start Capture** to view real-time events.
2. Go to the **Logs** tab and search for any ingestion errors or warnings.
3. Open the **Monitoring** page and review events/bytes in and out over time for this Source.

**Scenario:** Your upstream OpenTelemetry Collector logs show:
```
Exporting failed. Will retry the request after interval
```

**Checkpoint question:** What setting should you adjust first, and why?

<details>
<summary>Answer</summary>
Increase **Advanced Settings** > **Keep-alive timeout (seconds)** so the Collector has enough time to send its events before the connection's keep-alive period elapses.
</details>

---

## Part 9: Review Internal Fields

Inspect captured events (from Part 3 or Part 7) for these metadata fields:

| Field | Purpose |
|---|---|
| `__otlp.version` | OTLP version used to parse the event (`0.10.0` or `1.3.1`) |
| `__otlp.type` | Signal type: `logs`, `metrics`, or `traces` |
| `__otlp.extracted` | `true` if the event was extracted from a batch (per span/data point/log record) |

**Checkpoint question:** You want a Pipeline to apply different logic to trace data versus log data. Which internal field would you filter on?

<details>
<summary>Answer</summary>
`__otlp.type` — filter for `traces` vs. `logs`.
</details>

---

## Wrap-Up / Knowledge Check

1. Which Protobuf payload encoding is supported over HTTP, and which one is *not*?
2. Name two Advanced Settings that are available only when Protocol is set to `HTTP` (not shown for gRPC).
3. Why might you lower the **Requests-per-socket limit** setting?
4. What happens to Routes/Pipelines built for OTLP `0.10.0` if you switch a Source to `1.3.1`?
5. What's a key advantage of using HTTP over gRPC when multiple applications need to send data through one Source?

<details>
<summary>Answers</summary>

1. Binary Protobuf is supported over HTTP; JSON Protobuf is **not** supported.
2. Any two of: **Health check endpoint**, **Active request limit (on-prem only)**, **Requests-per-socket limit**, **Socket timeout**, **Request timeout**, **Keep-alive timeout**, **IP allowlist regex**, **IP denylist regex**. (gRPC only exposes **Active connection limit** and **Environment**.)
3. To force clients to distribute requests across more connections/Worker Node processes when connections would otherwise stay open a long time (60+ seconds) and concentrate load on a single process.
4. They may break due to breaking field-name changes introduced in OTLP `0.19.0`+; you must audit filters/logic for obsolete field names before upgrading.
5. HTTP supports **multiple auth tokens** on a single Source (with per-token Fields for sender metadata) — a capability gRPC does not offer — making it easier to onboard multiple senders with distinct credentials and tag their events for routing.

</details>

---

## Lab Complete

You've configured an HTTP-based OTel Source end-to-end: General Settings and extraction, multi-tenant Authentication, TLS, Persistent Queue, HTTP-specific Advanced Settings, and troubleshooting. As a next step, consider connecting this Source to a Route or QuickConnect Destination and building a Pipeline that filters on `__otlp.type`.

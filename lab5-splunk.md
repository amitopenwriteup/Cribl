# Cribl Stream Integration Guide: Splunk Cloud

This guide covers connecting Cribl Stream to **Splunk Cloud** — authentication, connection setup, and end-to-end validation.

---

## 1. Overview

```
Sources ──▶ Cribl Stream (Routes / Pipelines) ──▶ Splunk Cloud (via HEC)
```

The **Splunk HEC Destination** is the recommended way to send data from Cribl Stream to Splunk Cloud. It streams data over HTTPS directly into Splunk's indexing pipeline.

---

## 2. Prerequisites

- A Splunk Cloud instance with HEC enabled
- Admin access to create HEC tokens
- Network path from your Cribl Workers to your Splunk Cloud HEC endpoint (typically port 8088)

---

## 3. Create a HEC Token in Splunk Cloud

1. In Splunk Cloud, go to **Settings > Add Data > Monitor > HTTP Event Collector** (or **Settings > Data Inputs > HTTP Event Collector**).
2. Click **New Token**, name it (e.g. `cribl-hec`).
3. Choose a **source type** — Automatic, unless the token is dedicated to a specific type.
4. Select the **allowed indexes** for this token.
5. Save. Splunk generates a token GUID — copy it for use in Cribl.
6. Confirm the HEC endpoint is reachable — Splunk Cloud typically exposes it at:
   ```
   https://<your-stack>.splunkcloud.com:8088
   ```
   Always use **HTTPS**; the endpoint defaults to HTTP if not specified explicitly.

---

## 4. Configure the Splunk HEC Destination in Cribl Stream

1. Go to **Data > Destinations** (or **Routing > QuickConnect > Add Destination**).
2. Select **Splunk > HEC**, then **Add Destination**.
3. Configure:
   - **Output ID** – unique name, e.g. `splunk-cloud-hec`
   - **Splunk HEC endpoint(s)** – your `https://<stack>.splunkcloud.com:8088` URL
   - **Authentication method**:
     - **Manual** – paste the HEC token directly
     - **Secret** – reference a stored secret (recommended for production)
4. Under **TLS**, enable certificate validation.

### 4.1 Optional: SSL setup via Universal Forwarder credentials

If you need mutual TLS or are integrating via the Splunk Cloud Universal Forwarder credentials package:

1. In Splunk Cloud, download the **Universal Forwarder credentials app** to your desktop.
2. Rename the file suffix from `.spl` to `.tar.gz`, then untar/unzip it.
3. Locate the certificate files inside (`server.pem`, etc.) — you'll need these in Cribl.
4. In Cribl, go to **Settings > Global Settings > Security > Certificates**.
5. Populate:
   - **Certificate** – drag/drop `server.pem`
   - **Private Key** – paste just the private key section from `server.pem`

5. Tune **Advanced Settings** if needed (Max body size, Request concurrency, Flush period).
6. Click **Save**, then **Commit & Deploy**.

---

## 5. Validating End-to-End Data Flow

### 5.1 Destination-level test

- Open the Splunk HEC destination config, go to the **Test** tab, click **Run Test**. You should see a **Success** message.

### 5.2 Manual curl test

```bash
curl -k "https://<your-stack>.splunkcloud.com:8088/services/collector" \
  -H "Authorization: Splunk <hec-token>" \
  -d '{"event": "cribl test event", "sourcetype": "manual"}'
```

### 5.3 Verify in Splunk Cloud

1. In Splunk Cloud, search:
   ```
   index=main cribl_pipe=* OR "cribl test event"
   ```
2. Confirm the event appears with the expected index, sourcetype, and timestamp.

### 5.4 Full pipeline check

- Route a live Source through Cribl Stream to the Splunk Cloud destination.
- Compare event counts in Splunk against Cribl Stream's **Monitoring** dashboard (events in/out per destination) to confirm no data loss.
- Check the destination's **persistent queue (PQ)** status — a growing PQ indicates backpressure or connectivity issues to Splunk Cloud.

---

## 6. Troubleshooting Checklist

| Symptom | Likely Cause |
|---|---|
| Destination test fails | Wrong endpoint/port, firewall blocking outbound HTTPS, invalid token |
| No events in Splunk | Token's allowed indexes don't match search index, HEC disabled, wrong source type |
| TLS/SSL handshake errors | Missing or mismatched certs, endpoint using HTTP instead of HTTPS |
| Events delayed or dropped | PQ filling up, backpressure set to "drop", network latency to Splunk Cloud |

---

*Compiled from Cribl documentation as of August 2026. Steps may vary slightly by Cribl Stream version.*

# Lab: Setting Up Cribl Stream Leader and Worker Nodes

**Based on:** [Set Up Leader and Worker Nodes — Cribl Docs](https://docs.cribl.io/stream/setting-up-leader-and-worker-nodes/)

## Goal

Stand up a small Distributed deployment of Cribl Stream: one **Leader Node** and two **Worker Nodes**, connect the Workers to the Leader, and verify the deployment from the Leader UI. This lab uses Docker Compose so it's disposable and repeatable on a laptop.

## Prerequisites

- Docker and Docker Compose installed
- ~2 GB free RAM, ports `9000`, `9001`, `9002`, `4200` free on your host
- Basic familiarity with a terminal

## Lab Architecture

```
                 ┌──────────────────────┐
                 │   Leader Node         │
                 │  (cribl-leader)       │
                 │  UI: :9000            │
                 │  Distributed API:4200 │
                 └──────────┬────────────┘
                             │  CRIBL_DIST_LEADER_URL
             ┌───────────────┴───────────────┐
             ▼                                 ▼
   ┌────────────────────┐          ┌────────────────────┐
   │  Worker Node 1      │          │  Worker Node 2      │
   │  (cribl-worker-1)   │          │  (cribl-worker-2)   │
   │  UI: :9001          │          │  UI: :9002          │
   └────────────────────┘          └────────────────────┘
```

---

## Step 1 — Create the lab directory and Compose file

```bash
mkdir -p cribl-lab && cd cribl-lab
```

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  cribl-leader:
    image: cribl/cribl:latest
    container_name: cribl-leader
    hostname: cribl-leader
    ports:
      - "9000:9000"   # Leader UI
      - "4200:4200"   # Distributed API (Workers connect here)
    environment:
      - CRIBL_DIST_MODE=leader
      - CRIBL_DIST_LEADER_URL=tcp://criblmaster@0.0.0.0:4200
    networks:
      - cribl-lab

  cribl-worker-1:
    image: cribl/cribl:latest
    container_name: cribl-worker-1
    hostname: cribl-worker-1
    ports:
      - "9001:9000"   # Worker UI (teleport target)
    environment:
      - CRIBL_DIST_LEADER_URL=tcp://criblmaster@cribl-leader:4200
    depends_on:
      - cribl-leader
    networks:
      - cribl-lab

  cribl-worker-2:
    image: cribl/cribl:latest
    container_name: cribl-worker-2
    hostname: cribl-worker-2
    ports:
      - "9002:9000"   # Worker UI (teleport target)
    environment:
      - CRIBL_DIST_LEADER_URL=tcp://criblmaster@cribl-leader:4200
    depends_on:
      - cribl-leader
    networks:
      - cribl-lab

networks:
  cribl-lab:
    driver: bridge
```

> This mirrors the docs' **"Set Mode with Environment Variables"** section: `CRIBL_DIST_MODE` picks `leader`/`worker`, and `CRIBL_DIST_LEADER_URL` points a Worker at the Leader. If `CRIBL_DIST_LEADER_URL` is set without `CRIBL_DIST_MODE`, the instance defaults to `worker` — that's why the two worker services only set the leader URL.

## Step 2 — Start the Leader first

```bash
docker compose up -d cribl-leader
docker compose logs -f cribl-leader
```

Wait until you see a line indicating the Leader has started and is listening on port `4200`. Then `Ctrl+C` out of the log stream (the container keeps running).

## Step 3 — Start the Workers

```bash
docker compose up -d cribl-worker-1 cribl-worker-2
docker compose logs -f cribl-worker-1 cribl-worker-2
```

Look for log lines confirming each Worker registered with the Leader at `cribl-leader:4200`.

## Step 4 — Log in to the Leader UI

1. Open **http://localhost:9000**
2. Default credentials are `admin` / `admin` on first boot (you'll be prompted to change the password)
3. In the sidebar, go to **Settings → Global → Distributed Settings → General Settings** and confirm **Mode** shows this instance as **Leader** — matching what the docs describe under **"Set Mode with UI Settings"**

## Step 5 — Verify the Workers registered

1. In the sidebar, select **Workers**
2. You should see `cribl-worker-1` and `cribl-worker-2` listed and reporting healthy
3. Click into one Worker to **teleport** into its authenticated UI — a purple border appears around the UI confirming you're viewing the Worker remotely (per the docs' **"UI Access to Workers (Teleporting)"** section)
4. Close out with the **X** on the purple border to return to the Leader's Workers page

## Step 6 — (Optional) Tag a Worker via Teleport

1. Teleport into `cribl-worker-1`
2. Go to **Worker Settings → Distributed Settings**
3. Add a tag, e.g. `lab-worker-1`, and **Save**
4. Accept the confirmation — tags can change which Worker Group the Leader maps this Worker into

## Step 7 — Commit and deploy a trivial change

1. On the Leader, go to **Routes** (or any config page) and make a small change — e.g. rename the default Route description
2. Use **Commit & Deploy** in the top bar
3. Watch the change roll out to both Workers — this exercises the config-bundle push path the docs reference under **Manage Config Bundles / Configurations and Restart**

---

## Alternative: Set mode via CLI instead of environment variables

If you'd rather practice the CLI approach from the docs (`mode-master` / `mode-worker`), exec into a plain container without the env vars pre-set:

```bash
docker exec -it cribl-leader ./cribl mode-master -p 4200
docker exec -it cribl-worker-1 ./cribl mode-worker -H cribl-leader -p 4200
```

Note: env vars take priority over CLI/UI settings, so remove `CRIBL_DIST_*` from the Compose file first if you want the CLI commands to actually take effect.

## Things to try next

- Configure **Leader High Availability** with a standby Leader (docs: *Leader High Availability/Failover*)
- Edit `instance.yml` directly instead of using env vars, to see the `distributed.mode: master` / `distributed.mode: worker` structure the docs show
- Rotate the auth token following *How to Secure the Auth Token for the Leader Node*
- Map Workers into named **Worker Groups** using tags instead of the default group

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Worker never appears on Leader's **Workers** page | Check `cribl-worker-*` can resolve/reach `cribl-leader:4200` — confirm all three services are on the `cribl-lab` network |
| Leader UI won't load on :9000 | Give it another 15–20s on first boot; check `docker compose logs cribl-leader` |
| "CRIBL_DIST_* variable is defined" blocks UI mode settings | Expected — the docs note env vars take priority over UI-based mode configuration; unset them to use the UI instead |

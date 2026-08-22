# Lab: Worker Groups, Worker Deployment, and Sizing — Self-Hosted (Rocky Linux)

**Based on official Cribl Docs:**
- [OS and System Requirements](https://docs.cribl.io/stream/requirements/)
- [Set Up Leader and Worker Nodes](https://docs.cribl.io/stream/setting-up-leader-and-worker-nodes/)
- [Manage Worker Groups](https://docs.cribl.io/stream/manage-worker-groups/)
- [Bootstrap Workers from Leader](https://docs.cribl.io/stream/deploy-workers/)
- [Sizing and Scaling](https://docs.cribl.io/stream/scaling/)
- [Run Cribl Stream](https://docs.cribl.io/stream/run-stream/)
- [Running Cribl Stream on a Hardened OS](https://docs.cribl.io/stream/usecase-rhel8-stig/)
- [Cribl.Cloud vs. Self-Hosted](https://docs.cribl.io/stream/cloud-vs-self-hosted/)

> **This version replaces the Cribl.Cloud-specific steps** (Cloud portal, Sizing Calculator UI, deprovisioning) with the equivalent **self-hosted, distributed deployment** steps on **Rocky Linux**. Rocky Linux is a RHEL-derivative, so it falls under Cribl's supported **Red Hat family** for the bootstrap script and general OS requirements.

---

## Overview

This lab walks through standing up a **self-hosted, distributed Cribl Stream deployment** on Rocky Linux: a Leader Node, one or more Worker Groups, and Worker Nodes that bootstrap and register themselves against your own Leader — no Cribl.Cloud involved.

---

## Learning Objectives

By the end of this lab, you will be able to:

- Confirm your Rocky Linux hosts meet Cribl Stream's OS/system requirements
- Set up a Leader Node and create Worker Groups from the on-prem UI
- Deploy and register Worker Nodes on Rocky Linux using the Leader's bootstrap API
- Use tags to auto-assign a bootstrapped Worker to a specific Worker Group
- Calculate the number of cores and Worker Nodes needed for a given data volume
- Configure the Worker Process count appropriately for your hardware
- Identify the extra responsibilities (security, patching, Git, hardening) that come with self-hosting, and the features you gain over a pure Cribl-managed Cloud deployment

---

## Prerequisites

- One or more Rocky Linux hosts (Leader + Worker Nodes), meeting:
  - 64-bit Linux kernel **≥ 3.10**, glibc **≥ 2.17** (any current Rocky Linux release satisfies this)
  - **git ≥ 1.8.3.1** installed locally on the **Leader** host (configuration changes are committed to git before deployment)
  - Root or sudo access, or a plan to run Cribl Stream as a non-root user (see §2.5)
- Network/firewall access between hosts:
  - Leader ↔ Worker: **port 4200** (outbound from Worker to Leader, ongoing)
  - Worker → `https://cdn.cribl.io`: **port 443** (during install)
  - Worker → on-prem Leader: **port 9000** (during bootstrap)
- Minimum single-node hardware guidance (scale up from here per Part 4): **4 physical cores**, **8 GB RAM** beyond OS overhead, **5 GB free disk** (more with persistent queuing enabled)

---

## Estimated Time

45–60 minutes

---

# Part 1: Set Up the Leader and Create Worker Groups

## 1.1 Background

In a self-hosted distributed deployment, **you** run and manage the Leader Node — there's no Cribl.Cloud portal. The Leader is the single source of truth for configuration, and it pushes that configuration out to Worker Nodes organized into **Worker Groups**.

## 1.2 Install the Leader on Rocky Linux

1. Download the Cribl Stream `.tgz` package for Linux from Cribl's Downloads page (or use the `curl` command from the Downloads page).
2. **If SELinux is enabled** (Rocky Linux ships with SELinux in `Enforcing` mode by default): download and un-tar the package **directly into `/opt/`**. This avoids SELinux context/labeling problems that can occur when a file is moved into `/opt/` after being downloaded elsewhere.

   ```shell
   cd /opt
   curl <cribl-download-url> -o cribl.tgz
   tar xvzf cribl.tgz
   ```

3. Set the deployment mode to Leader:

   ```shell
   cd /opt/cribl/bin
   ./cribl mode-master
   ```

4. Start Cribl Stream:

   ```shell
   ./cribl start
   ```

5. Log into the Leader's web UI (default port **9000**), and complete the initial registration/password-change prompt.

> **SELinux note:** If you plan to run in `Enforcing` mode long-term (recommended for hardened/STIG environments), review **Running Cribl Stream on a Hardened OS** and the **SELinux (Enforcing Mode) Configuration** doc before going to production — you may need custom policy modules for Cribl Stream's ports and file contexts.

## 1.3 Create a Worker Group (On-Prem UI)

Unlike Cribl.Cloud, there's no "provider/region" picker or Sizing Calculator wizard — you define the Group, then decide how many Worker Nodes to point at it (manually, or via automation like Ansible/Terraform on your own infrastructure).

1. In the Leader's UI, go to **Manage Worker Groups** (top nav, or via **Groups** in the sidebar).
2. Select **Add Worker Group**.
3. Enter a **Group name** (this is what you'll reference in Mapping Rules and bootstrap `tag`/`group` parameters later).
4. Optionally add a **Description**.
5. Select **Save**, then **Commit & Deploy** to push the new Group's baseline config.

> There's no artificial cap on the number of Worker Groups you can create on a self-hosted Leader (beyond what your license and hardware support) — unlike the Cribl.Cloud default of 10 per Workspace.

## 1.4 Sizing a Worker Group (Manual, Self-Hosted)

Since there's no built-in Sizing Calculator on a self-hosted Leader, use the manual sizing math in **Part 3** of this lab to decide:

- How many Rocky Linux hosts to provision as Workers for this Group
- How many vCPUs/cores each host needs
- The `Process count` setting for the Group

## Checkpoint: Part 1

- [ ] Leader is installed under `/opt/cribl`, running in `mode-master`, reachable on port 9000
- [ ] I created at least one Worker Group and committed/deployed it
- [ ] I understand that self-hosted sizing is manual — no in-app calculator

---

# Part 2: Deploying & Registering Workers on Rocky Linux

## 2.1 Background: Bootstrap from the Leader

Rocky Linux is part of Cribl's supported **Red Hat family** (RHEL, CentOS, Rocky, Amazon Linux) for the bootstrap script. A Worker host with **no Cribl Stream pre-installed** can fully provision itself by running a script served directly from the Leader.

The Leader exposes:

```
GET http://<leader hostname or IP>:9000/init/install-worker.sh
```

## 2.2 Requirements Before You Start

Confirm on each Worker host:

- Outbound **port 4200** to the Leader (ongoing, after install)
- Outbound **port 443** to `https://cdn.cribl.io` (during install)
- Outbound **port 9000** to your on-prem Leader (during bootstrap)
- If your Rocky Linux hosts run **firewalld** (default on Rocky), confirm these outbound rules aren't blocked. Outbound traffic is typically allowed by default firewalld zones, but verify if you have custom egress rules:

  ```shell
  firewall-cmd --list-all
  ```

- If traffic must transit a proxy, configure **System Proxy** first (**Settings > Global > System > System Proxy Configuration**, on-prem only).

## 2.3 Step-by-Step: Bootstrap a New Rocky Linux Worker (UI Method)

1. In the Leader's sidebar, select **Workers**.
2. Select **Add/Update Worker Node**.
3. Select **Linux > Add**.
4. Leave **Install package location** as `Cribl CDN`, or switch to `Download URL` if this Rocky Linux host can't reach the public CDN (e.g., air-gapped/restricted network) and you have an internal mirror.
5. Confirm the **Leader hostname/IP** field points to this Leader.
6. Set the target **Group** (from Part 1), **User**, and **User Group** — see §2.5 for non-root considerations.
7. Copy the generated script.
8. SSH into the Rocky Linux Worker host and paste/run the script.

## 2.4 Step-by-Step: Bootstrap via curl (CLI Method)

**Fetch and execute in one step**, as root or with sudo:

```shell
curl http://<leader hostname or IP>:9000/init/install-worker.sh?token=<auth-token> | sudo bash -
```

> If you hit `ssl certificate problem: self signed certificate in certificate chain` (common with a self-signed Leader cert in a self-hosted lab), add `-k` to skip validation — only acceptable for lab/test use, not production.

### Try it yourself

Your Leader is `stream-leader.internal.corp`, token is `r0cky123`, and you want this new Rocky Linux host bootstrapped immediately as root. Write the command.

<details>
<summary>Answer</summary>

```shell
curl http://stream-leader.internal.corp:9000/init/install-worker.sh?token=r0cky123 | bash -
```

(No `sudo` needed if you're already root; add `sudo` before `bash -` if running as a regular user with sudo rights.)
</details>

## 2.5 Running as a Non-Root User on Rocky Linux

Cribl Stream doesn't require root to run day-to-day, but binding to privileged ports (< 1024) does. On Rocky Linux, grant the capability directly to the binary instead of running as root:

```shell
setcap cap_net_bind_service=+ep /opt/cribl/bin/cribl
```

> On some RHEL-family systems (Rocky included), you may need the `-i` flag:
>
> ```shell
> setcap -i cap_net_bind_service=+ep /opt/cribl/bin/cribl
> ```

**Remember:** every Cribl Stream **upgrade** replaces the binary, which strips this capability — re-run `setcap` after each upgrade, or automate it in your upgrade playbook.

If you see `bind EACCES 0.0.0.0:<port>` in the Worker/API logs, `setcap` most likely didn't run successfully — check it first.

## 2.6 Assigning a Worker to a Group with Tags

Exactly as in the Cloud version of this lab: use `tag` on the bootstrap URL plus a **Mapping Ruleset** filter on the Leader to route new Rocky Linux Workers into the right Group automatically — useful when you're standing up many hosts via a config management tool (Ansible, Puppet, cloud-init, etc.).

```shell
curl "http://<leader>:9000/init/install-worker.sh?tag=rockylinux&tag=dc1&token=<token>" | sudo bash -
```

Matching Mapping Ruleset filter:

```shell
cribl.tags.includes('rockylinux') && cribl.tags.includes('dc1')
```

## 2.7 Key API Query Parameters Reference

| Parameter | Required? | Purpose |
|---|---|---|
| `token` | optional | Leader's shared secret (`authToken`) |
| `group` | optional | Target Worker Group; defaults to `default` |
| `download_url` | optional | Internal mirror, for air-gapped Rocky Linux hosts that can't reach the public CDN |
| `tag` | optional | Used with Mapping Rules to route the Worker to a Group; repeatable |
| `user` | optional | User to run Cribl Stream as; defaults to `cribl` |
| `user_group` | optional | Owning user group; defaults to `cribl` |
| `install_dir` | optional | Install path; defaults to `/opt/cribl` |

## Checkpoint: Part 2

- [ ] I confirmed firewalld / egress rules allow ports 4200, 443, and 9000 as needed
- [ ] I bootstrapped a Worker via `curl | bash` against my own Leader
- [ ] I applied `setcap` for non-root privileged-port binding, and know to redo it after upgrades
- [ ] I can write a tagged bootstrap URL that maps to a specific Worker Group

---

# Part 3: Sizing and Scaling Considerations

*(This section is deployment-agnostic — the math is identical whether the Worker Nodes are on Rocky Linux VMs, bare metal, or Cribl-managed cloud infrastructure.)*

## 3.1 Two Ways to Scale

- **Scale Up** — more resources within a single instance (more Worker Processes per Rocky Linux host)
- **Scale Out** — more Worker Nodes across your distributed deployment

## 3.2 Scaling Up: Worker Process Count

Set per Worker Group (**Group Settings > Worker Processes**):

- Positive number = absolute number of Worker Processes
- Negative number = relative to CPU count (e.g., `-2` means *CPUs minus 2*)
- Default: **`-2`**

| Rocky Linux Host Profile | Recommended Process Count | Why |
|---|---|---|
| x86_64, hyperthreading enabled (typical bare metal/VM) | `-2` (default) | Reserves headroom for OS + Cribl API overhead |
| Non-hyperthreaded (e.g., ARM-based Rocky Linux hosts) | `-1` | Reserves 1 physical core, maximizes usable processes |
| Syslog Source Load Balancer enabled | Reduce effective count by `1` more | Reserves a core for the LB process |

### Try it yourself

A Rocky Linux Worker host has 8 physical cores with hyperthreading (16 vCPUs), `Process count = -2`. How many Worker Processes spawn?

<details>
<summary>Answer</summary>

**14 processes** (16 vCPUs − 2 = 14).
</details>

## 3.3 Scaling Out: Core Sizing Math

- **Intel/AMD (x86_64) with hyperthreading:** plan for **~200 GB/day per vCPU**, i.e., **400 GB/day per physical core**.
- **ARM64 (e.g., Ampere-based Rocky Linux hosts):** 1 physical core = 1 vCPU, ~20% higher throughput → **480 GB/day per vCPU**.

### Worked Example

> 100 GB IN → 100 GB OUT to each of 3 Destinations = 400 GB total = **1 physical core** (2 vCPUs) on x86_64.

### Try it yourself

You expect 3 TB/day IN and 1 TB/day OUT (single Destination) on standard x86_64 Rocky Linux hosts. How many physical cores do you need (400 GB/day/core guideline)?

<details>
<summary>Answer</summary>

Total = 4 TB/day = 4096 GB/day.
4096 ÷ 400 ≈ 10.24 → **round up to 11 physical cores** (22 vCPUs) — always round up and add headroom.
</details>

## 3.4 Estimating Number of Nodes

- Minimum per Node: **8 x86_64 vCPUs** (4 cores hyperthreaded) — below this, OS overhead eats too much capacity
- Maximum per Node: **48 x86_64 vCPUs** (24 cores hyperthreaded) — keeps disk I/O manageable with persistent queueing active
- Always **plan to handle peak load with 20% of Worker Nodes offline** (patching, restarts — which, on self-hosted Rocky Linux, is entirely your responsibility; see Part 4)
- For 5–20 TB/day: size for **4–8 Worker Nodes per Worker Group**

### Worked Example (from Cribl's guidance, x86_64 sizing)

**Scenario:** 6 TB/day IN, 10 TB/day OUT → 16 TB/day total, on 16-vCPU x86_64 Rocky Linux hosts.

- Usable vCPUs per host ≈ 14 (2 reserved for OS/overhead) × 200 GB/day = 2800 GB/day ≈ **2.8 TB/day per Node**
- 16 TB ÷ 2.8 TB ≈ 5.7 → round up → **6 Nodes** for peak load
- +20% redundancy (20% of 6 ≈ 1.2, round up) → **7 Nodes total**

### Try it yourself

Your on-prem deployment needs 8 TB/day total (IN+OUT) on 16-vCPU x86_64 Rocky Linux hosts (~2.8 TB/day per Node, per above). How many Nodes for peak load, and how many including the 20% buffer?

<details>
<summary>Answer</summary>

8 ÷ 2.8 ≈ 2.86 → round up → **3 Nodes** for peak load.
20% of 3 = 0.6 → round up → **1 additional Node** → **4 Nodes total**.
</details>

## 3.5 Memory Considerations

- Default heap: **2048 MB (2 GB) per Worker Process** — increase only if you hit limits.
- Consumed by lookups (size scales with lookup size) and stateful Functions (**Aggregations**, **Dynamic Sampling**, **Suppress**), proportional to throughput and cardinality.
- **External memory** (outside the heap limit) is mainly consumed by Destinations, especially load-balanced ones — plan extra headroom on your Rocky Linux hosts' total RAM beyond the heap setting.

## Checkpoint: Part 3

- [ ] I can compute Worker Processes spawned from vCPU count and `Process count`
- [ ] I can estimate physical cores from a GB/day figure using the x86_64 or ARM64 guideline
- [ ] I can walk through a full Node-count sizing example including the 20% offline buffer
- [ ] I've sized headroom for both heap and external memory on my Rocky Linux hosts

---

# Part 4: What Self-Hosting on Rocky Linux Means for You

Since you're self-hosting rather than using Cribl.Cloud, these responsibilities and capabilities apply to you directly. This section reframes the Cloud-vs-self-hosted comparison around **what you now own**.

## 4.1 Responsibilities You Now Own

| Area | Your Responsibility on Self-Hosted Rocky Linux |
|---|---|
| Network isolation & physical/host access | You control firewalld rules, SELinux policy, SSH access, VM/bare-metal hardening |
| Configuration & token security | You secure `.tgz` install paths, `authToken` values, and config files on disk |
| Data at rest | You configure disk encryption (e.g., LUKS) if required — not automatic |
| Data in motion | You configure TLS between Leader/Workers and to Sources/Destinations yourself |
| Patch management | You patch both the Rocky Linux OS **and** Cribl Stream itself, and re-run `setcap` after upgrades |
| Restarts/upgrades | Manual, via **Settings > Controls**, or your own automation (Ansible/systemd) |
| Compliance posture | You own your own SOC 2 / GDPR / STIG compliance work — Cribl's cloud attestations don't cover your infrastructure |

## 4.2 Features You Gain by Self-Hosting

These are **unavailable on pure Cribl-managed Cloud Workers**, but available to you now:

- **Script Collector** and **File System Collector**
- **Filesystem Destination**, including staging-directory support in file-based Destinations
- **Exec Source**
- **System State Source**
- **Settings > Global > Scripts** — run custom shell scripts directly from the Leader
- Full **Git remote repository** support under **Settings > Global > System > Git Settings** (Cribl.Cloud only offers a preconfigured local git client, no remotes)
- Unrestricted **Persistent Queue** sizing — you define the **Queue size limit** based on your own provisioned disk, rather than the 1 GB/process cap Cribl enforces on its own managed Workers
- Full **Licensing** page and unlimited Worker Groups (license permitting) — no Workspace cap of 10

## 4.3 Hardening Checklist for Rocky Linux

Before going to production:

- [ ] Confirm SELinux mode (`getenforce`) and, if `Enforcing`, review **Running Cribl Stream on a Hardened OS** for required policy adjustments
- [ ] Review `firewalld` zones/rules on both Leader and Worker hosts for the required ports (4200, 9000, 443, plus any Source/Destination-specific ports)
- [ ] Apply `setcap cap_net_bind_service=+ep` if running as non-root, and script it into your upgrade process
- [ ] Confirm git is installed and reachable on the Leader (`git --version`, must be ≥ 1.8.3.1)
- [ ] Decide on your patch/upgrade cadence and who owns it operationally
- [ ] Decide whether you need Git **remote** repository integration for config version control

## Checkpoint: Part 4

- [ ] I can list at least 3 responsibilities I now own that Cribl.Cloud would otherwise handle
- [ ] I can list at least 3 features I gained by choosing self-hosted over Cribl-managed Cloud
- [ ] I've walked through the Rocky Linux hardening checklist

---

## Final Knowledge Check

1. Why should you download and un-tar the Cribl Stream package directly into `/opt/` on an SELinux-enforcing Rocky Linux host?
2. Which three ports matter most during and after Worker bootstrap?
3. What command grants the Cribl Stream binary permission to bind privileged ports without running as root, and when must you re-run it?
4. Name two features available to you now that would NOT be available on a pure Cribl-managed Cloud Worker Group.
5. True or False: On self-hosted Rocky Linux, Cribl Stream itself patches the underlying OS for you.

<details>
<summary>Answers</summary>

1. To avoid SELinux file-context/labeling issues that can occur when a file is downloaded elsewhere and then moved into `/opt/`.
2. Port **4200** (ongoing Worker-to-Leader), port **443** (to `cdn.cribl.io` during install), and port **9000** (to the on-prem Leader during bootstrap).
3. `setcap cap_net_bind_service=+ep /opt/cribl/bin/cribl` (add `-i` on some RHEL-family systems) — re-run after every Cribl Stream upgrade, since upgrading replaces the binary and strips the capability.
4. Any two of: Script Collector, File System Collector, Filesystem Destination, Exec Source, System State Source, `Settings > Global > Scripts`, Git remote repos, unrestricted Persistent Queue sizing.
5. **False** — on self-hosted deployments, you patch both the Rocky Linux OS and the Cribl Stream application yourself.
</details>

---

## Next Steps

- Continue to **Sources, Routes, Pipelines, and Functions** to start processing data through your new self-hosted Worker Group.
- Review **[High Availability Requirements](https://docs.cribl.io/stream/deploy-ha-requirements/)** if you need a standby Leader for resilience.
- Review **[SELinux (Enforcing Mode) Configuration](https://docs.cribl.io/stream/usecase-rhel8-stig/)** in full before hardening a production Rocky Linux deployment.

## Additional Resources

- [OS and System Requirements (source doc)](https://docs.cribl.io/stream/requirements/)
- [Set Up Leader and Worker Nodes (source doc)](https://docs.cribl.io/stream/setting-up-leader-and-worker-nodes/)
- [Manage Worker Groups (source doc)](https://docs.cribl.io/stream/manage-worker-groups/)
- [Bootstrap Workers from Leader (source doc)](https://docs.cribl.io/stream/deploy-workers/)
- [Sizing and Scaling (source doc)](https://docs.cribl.io/stream/scaling/)
- [Run Cribl Stream / Non-Root User Configuration (source doc)](https://docs.cribl.io/stream/run-stream/)
- [Running Cribl Stream on a Hardened OS (source doc)](https://docs.cribl.io/stream/usecase-rhel8-stig/)
- [Cribl.Cloud vs. Self-Hosted (source doc)](https://docs.cribl.io/stream/cloud-vs-self-hosted/)

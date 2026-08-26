# Lab: Deploying Cribl Edge on Linux

**Source reference:** [Install Cribl Edge on Linux](https://docs.cribl.io/edge/deploy-linux.md)

## Objectives

By the end of this lab, you will be able to:

- Verify a Linux host meets the minimum requirements for Cribl Edge
- Install Cribl Edge as a standalone node (`mode-edge`)
- Connect a Cribl Edge node to a Leader (`mode-managed-edge`)
- Configure Cribl Edge to start on boot as a service
- Run Cribl Edge and Cribl Stream collocated on the same host
- Run multiple Cribl Edge instances on the same host
- Troubleshoot a basic deployment

**Estimated time:** 45–60 minutes

**Prerequisites:**
- A Linux VM/host (or container) with sudo access
- Downloaded Cribl install package (`cribl-<version>-<build>-<arch>.tgz`)
- (Optional, for managed-edge exercises) Access to a Cribl Leader node and its auth token

---

## Part 1 — Check Minimum Requirements

1. Check your kernel version (must be a 64-bit kernel >= 3.10):

   ```shell
   uname -r
   ```

2. Check your glibc version (must be >= 2.17):

   ```shell
   ldd --version
   ```

3. Check available disk space (need 5 GB free per volume for operational purposes):

   ```shell
   df -h
   ```

4. Check CPU and RAM (need ~1 GHz processor, 1024 MB RAM minimum):

   ```shell
   nproc
   free -m
   ```

**Checkpoint:** Record your kernel version, glibc version, and available disk space. Do they meet the minimums?

---

## Part 2 — Download and Unpack Cribl Edge

1. Download the install package from [cribl.io/download](https://www.cribl.io/download) onto your Linux machine (or use a package already provided by your instructor).

2. Un-tar the package into `/opt/`, then rename the resulting directory to `cribl-edge`:

   ```shell
   cd /opt/
   tar xvzf cribl-<version>-<build>-<arch>.tgz
   mv /opt/cribl/ /opt/cribl-edge
   ```

   > **Note:** `/opt/cribl` and its subdirectories must reside on the same device. Don't mount separate devices inside `/opt/cribl`. For external storage (e.g., lookups, Persistent Queues), use a directory completely outside `/opt/cribl`.

3. Set `$CRIBL_HOME` for your session:

   ```shell
   export CRIBL_HOME=/opt/cribl-edge
   ```

**Checkpoint:** Run `echo $CRIBL_HOME` and confirm it prints `/opt/cribl-edge`.

---

## Part 3 — Install Cribl Edge as a Standalone Node

1. Change ownership of the installation to a non-privileged `cribl` user:

   ```shell
   chown -R cribl:cribl /opt/cribl-edge
   ```

2. Start the Cribl service:

   ```shell
   /opt/cribl-edge/bin/cribl start
   ```

3. Set the installation mode. For this exercise, since you don't have a Leader, use single-instance mode:

   ```shell
   /opt/cribl-edge/bin/cribl mode-edge
   ```

4. Configure the installation to start on boot and restart it:

   ```shell
   /opt/cribl-edge/bin/cribl boot-start enable -m systemd -u cribl
   sudo systemctl restart cribl-edge.service
   ```

5. Open a browser and go to `http://localhost:9420`. Log in with the default credentials:

   - Username: `admin`
   - Password: `admin`

**Checkpoint:** You should see the Cribl Edge UI. Change the default admin password before continuing.

---

## Part 4 — Connect to a Leader (Managed Edge)

> Skip this part if you don't have a Leader available; read through it to understand the workflow instead.

1. On your **Leader**, retrieve the auth token:
   **Settings** > **Global** > **System** > **Distributed Settings** > **Leader Settings** > **Auth token**

2. On the Edge node, run:

   ```shell
   /opt/cribl-edge/bin/cribl mode-managed-edge \
     -H <leader-hostname-or-IP> \
     -p <port> \
     -u <token> \
     [-g <fleet>]
   ```

3. Restart the service:

   ```shell
   sudo systemctl restart cribl-edge.service
   ```

**Discussion questions:**
- What happens if the token doesn't match the Leader's token?
- If you don't specify `-g <fleet>`, which Fleet is the node assigned to by default?
- What overrides a manually specified `-g <fleet>` value?

---

## Part 5 — Run Cribl Edge and Cribl Stream Collocated

1. Un-tar the install package twice, into two directories:

   ```shell
   cd /opt
   tar zxvf /tmp/cribl-<version>-<build>-<arch>.tgz
   mv cribl cribl-edge
   tar zxvf /tmp/cribl-<version>-<build>-<arch>.tgz
   ```

2. Set ownership for both installations:

   ```shell
   chown -R cribl:cribl /opt/cribl
   chown -R cribl:cribl /opt/cribl-edge
   ```

   > **Do not run Cribl Edge as root.** To listen on privileged ports (1–1024), grant the capability instead, via `override.conf`:
   > ```
   > [Service]
   > AmbientCapabilities=CAP_NET_BIND_SERVICE
   > ```
   > Add `CAP_DAC_READ_SEARCH` too if Edge needs to read protected files like `/var/log/*`.

3. Configure and start each product as its own service:

   ```shell
   # Cribl Edge
   /opt/cribl-edge/bin/cribl mode-managed-edge -H <leader-hostname-or-IP> -p <port>
   /opt/cribl-edge/bin/cribl boot-start enable
   sudo systemctl restart cribl-edge.service

   # Cribl Stream
   /opt/cribl/bin/cribl mode-worker -H <leader-hostname-or-IP> -p <port>
   /opt/cribl/bin/cribl boot-start enable
   sudo systemctl restart cribl.service
   ```

**Discussion questions:**
- What are the two distinct systemd service names used here, and why does each product need its own?
- What port does Cribl Edge's API listen on by default, and how does that differ from Cribl Stream?
- How would you make an Edge node listen on `0.0.0.0` instead of `127.0.0.1`?

---

## Part 6 — Run Multiple Instances of the Same Product

1. Un-tar the package and create two separate Edge directories:

   ```shell
   cd /opt
   tar zxvf /tmp/cribl-<version>-<build>-<arch>.tgz
   cp -r cribl/ cribl-edge-01/
   mv cribl/ cribl-edge-02/
   ```

2. Repeat the ownership, mode, and service steps from Part 3 **once per instance** — for `cribl-edge-01` and `cribl-edge-02`.

3. Ensure each instance runs on a dedicated port. Either:
   - Pass different ports on the `mode-managed-edge`/`mode-worker` command, **or**
   - Set the `CRIBL_AUTO_PORTS` environment variable to `1` on the host.

**Checkpoint:** Confirm both instances are running and reachable on distinct ports (`systemctl status cribl-edge-01.service`, etc., depending on how you named your services).

---

## Part 7 — Troubleshooting Checklist

Work through this checklist if a node isn't behaving as expected:

- [ ] Does `uname -r` and `ldd --version` meet the minimum OS requirements?
- [ ] Is `$CRIBL_HOME` set correctly for the shell/session you're using?
- [ ] Does `/opt/cribl-edge` (and subdirectories) live on a single device/mount?
- [ ] Is ownership set to the `cribl` user (not root)?
- [ ] Did you use the correct mode — `mode-edge` (standalone) vs. `mode-managed-edge` (Leader-connected)?
- [ ] Does the Leader token match exactly?
- [ ] Is the correct systemd service running — `cribl-edge.service` vs `cribl.service`?
- [ ] If collocated or running multiple instances, does each installation use a unique port?
- [ ] Can you reach the UI at `http://<host>:9420` (or your configured port)?

For deeper troubleshooting scenarios switching between Edge and Stream, see the Cribl University course: [Switching from Cribl Edge to Cribl Stream](https://university.cribl.io/switching-from-edge-to-stream).

---

## Lab Summary Table

| Task | Key Command |
|---|---|
| Unpack install | `tar xvzf cribl-<version>-<build>-<arch>.tgz` |
| Set ownership | `chown -R cribl:cribl /opt/cribl-edge` |
| Start service | `/opt/cribl-edge/bin/cribl start` |
| Standalone mode | `/opt/cribl-edge/bin/cribl mode-edge` |
| Managed mode | `/opt/cribl-edge/bin/cribl mode-managed-edge -H <host> -p <port> -u <token>` |
| Enable boot start | `/opt/cribl-edge/bin/cribl boot-start enable -m systemd -u cribl` |
| Restart service | `sudo systemctl restart cribl-edge.service` |
| Default UI | `http://localhost:9420` (admin/admin) |

---

## Knowledge Check

1. What is the minimum glibc version required to install Cribl Edge on Linux?
2. Why should Cribl Edge never be run as root, and what's the recommended alternative for binding to low ports?
3. When collocating Cribl Edge and Cribl Stream on one host, what two systemd service names are used?
4. What environment variable lets multiple instances of the same product auto-assign distinct ports?
5. What's the difference between `mode-edge` and `mode-managed-edge`?

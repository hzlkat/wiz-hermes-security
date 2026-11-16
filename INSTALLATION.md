# Wiz Deployment & Operations Guide (`dev-nvworkb01`)

This guide describes how to install, configure and verify the **Wiz Runtime Sensor** and the **Wiz Workload Scanner (Disk Scanner)** on the on-premise Ubuntu host `dev-nvworkb01`.

> **Recommended:** Use the Ansible playbook in [`ansible/`](ansible/README.md) for installation and credential rotation. It implements the steps below with the credentials stored in Ansible Vault. This guide describes the manual procedure and the verification commands.

> **Note:** The official Wiz documentation for non-cloud / on-prem hosts (docs.wiz.io) requires a tenant login. Values marked with ⚠️ should be checked against it before reuse.

---

## 0. Prerequisites

- Ubuntu host with systemd, `curl` and `sudo` rights
- Outbound HTTPS from the host to the Wiz backend (`*.wiz.io`). If a proxy is required, see `WIZ_HTTP_PROXY_URL` in section 1.
- A Wiz **service account** of type *Sensor* (Client ID + Client Secret) from the Wiz console
- Optional, for eBPF checks: `sudo apt install linux-tools-$(uname -r)` (provides `bpftool`)

---

## 1. Environment Configuration

The installer copies **every** `WIZ_*` environment variable into the sensor configuration (`/opt/wiz/sensor/host-store/engine.env`) and the disk scanner configuration (`/opt/wiz/disk-scanner/host-store/disk-scanner.env`). Therefore all values must be **exported** before running the installer.

```bash
# Enable the local Workload Scanner (Disk/SBOM Scanner)
export WIZ_ENABLE_DISK_SCANNER=1

# On-prem subscription & tags  ⚠️ check tag format against Wiz on-prem docs
export WIZ_SUBSCRIPTION_EXTERNAL_ID='dc-muc-vmware'
export WIZ_SUBSCRIPTION_NAME='dc-muc-vmware'
export WIZ_SUBSCRIPTION_TAGS='{"Region":"<region_value>","owner":"<owner_value>","vm_name":"dev-nvworkb01"}'

# Optional: only needed if the host must use a proxy
# export WIZ_HTTP_PROXY_URL='http://proxy.example.local:3128'
```

**Backend / region:** No region variable is needed. The installer authenticates via `auth.app.wiz.io` (default `WIZ_ENV=prod`), which applies to commercial tenants. The tenant's data center region is determined by the service account itself. Do **not** set a custom `WIZ_API_CLIENT_ENDPOINT` — the installer does not use it.

### Credentials

Do not write the secret directly into the command line: it would end up in the shell history and be visible in the process list (`ps`) while the installer runs. Read it interactively instead:

```bash
read -rp  "Wiz Client ID: "     WIZ_API_CLIENT_ID;     export WIZ_API_CLIENT_ID
read -rsp "Wiz Client Secret: " WIZ_API_CLIENT_SECRET; export WIZ_API_CLIENT_SECRET; echo
```

> The installer stores the credentials in `engine.env` (owner `root`, mode `400`). Treat the host accordingly and rotate the secret if the host is decommissioned.

---

## 2. Sensor & Scanner Installation

Run the installer with `sudo -E`, so the exported variables are passed through:

```bash
sudo -E bash -c "$(curl -fsSL https://downloads.wiz.io/sensor/sensor_install.sh)"
```

On Ubuntu, the installer performs a **package-based installation** (no container) and registers the systemd service `wiz-sensor`. The install log is written to `/opt/wiz/sensor/host-store/install.log`.

Afterwards remove the credentials from the current shell:

```bash
unset WIZ_API_CLIENT_ID WIZ_API_CLIENT_SECRET
```

---

## 3. Verification & Service Diagnostics

### 3.1 Service Status

```bash
# Service status
sudo systemctl status wiz-sensor.service --no-pager

# Last log entries of the service (add -f to follow live, stop with Ctrl+C)
sudo journalctl -u wiz-sensor.service -n 100 --no-pager

# Installer log
sudo tail -n 50 /opt/wiz/sensor/host-store/install.log

# Sensor log
sudo tail -n 50 /opt/wiz/sensor/host-store/sensor_logs/sensor.log

# Check that the subscription/tag variables were written to the configuration
sudo grep -E '^(SUBSCRIPTION|ENABLE_DISK_SCANNER)' /opt/wiz/sensor/host-store/engine.env
```

### 3.2 Verify eBPF Programs

```bash
# Lists loaded eBPF programs incl. owning process (if supported by the bpftool version)
sudo bpftool prog show | grep -i -B2 -A2 wiz
```

If the output is empty, this does not necessarily mean the sensor is inactive — older `bpftool` versions do not print the owning process. In that case rely on `systemctl status` and the sensor status in the Wiz console.

### 3.3 Verify the Workload Scanner

```bash
# Disk scanner configuration written by the installer
sudo ls -la /opt/wiz/disk-scanner/host-store/

# Local sensor host-store (SBOM artifacts, logs, configuration)
sudo ls -la /opt/wiz/sensor/host-store/
```

Expected content (excerpt; besides the directories below, the installer also creates `engine.env` and `install.log` here):

```text
drwx------ 2 root root  sboms
drwx------ 2 root root  sboms_partial
drwx------ 2 root root  sensor_logs
```

The final check is in the Wiz console: the host should appear in the inventory with installed packages and, after processing (see README, *Lessons Learned*), with vulnerability findings.

### 3.4 Test Runtime Detection

Runtime events are **not** written to the local journal — the sensor sends them to the Wiz backend. To generate activity:

```bash
# Run a Hermes agent prompt that spawns sub-processes / network connections
hermes chat -q "Check the network status of this host using shell commands"
```

Then check the result in the Wiz console: open the host `dev-nvworkb01` → runtime events / threats (menu names may vary by Wiz version). Processes started by the agent (e.g. `bash`, `curl`, `ss`) should be visible with the Hermes agent process (Python) as parent.

---

## 4. Troubleshooting

| Symptom | Check |
|---|---|
| Installer aborts with *Missing API key* | `WIZ_API_CLIENT_ID` / `WIZ_API_CLIENT_SECRET` not exported or `sudo` without `-E` |
| Installer fails at credential check | Outbound HTTPS to `auth.app.wiz.io` blocked → proxy/firewall; clock drift on the host (`timedatectl`) |
| Service running, host not visible in console | Wait for graph sync (15–60 min, see README); check subscription/tags in `engine.env` |
| No vulnerability findings | Disk scanner not enabled (`ENABLE_DISK_SCANNER` missing in `engine.env`) or SBOM still processing |

## 5. Reinstall with Changed Settings

To change subscription, tags or the disk scanner setting, export the variables again (section 1) and rerun the installer (section 2).

To make the installer exit instead when a sensor is already present (e.g. in automation), set `export WIZ_NO_REINSTALL=1`.

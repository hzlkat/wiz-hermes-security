# Wiz Deployment & Operations Guide (`dev-nvworkb01`)

This guide outlines the commands used to configure, install, and verify the Wiz Runtime Sensor and Workload Scanner on the On-Premise Debian host.

---

## 1. Environment Configuration

Set the environment variables that configure the local disk scanner, On-Premise subscription scope, and EU API endpoint. Update the tag values to match your environment:

```bash
# Enable the local Workload Scanner (Disk/SBOM Scanner)
WIZ_ENABLE_DISK_SCANNER=1

# Define On-Premise Subscription & Tag parameters
WIZ_SUBSCRIPTION_EXTERNAL_ID='dc-muc-vmware'
WIZ_SUBSCRIPTION_NAME='dc-muc-vmware'
WIZ_SUBSCRIPTION_TAGS='Region:<region_value>,owner:<owner_value>,vm_name:dev-nvworkb01'

# Set target Wiz API endpoint (EU Region)
WIZ_API_CLIENT_ENDPOINT='https://api.eu30.wiz.io'
```
The assignments above are used by the installation command below, which passes them explicitly through `sudo`.

## 2. Sensor & Scanner Installation
Execute the installation script using `sudo` to deploy the eBPF kernel sensor and background scanning daemon. The installer receives the client ID and secret as command-line arguments, so they may be visible to other users on the host while the installer runs. If your Wiz tenant's current installer documentation provides a safer credential method, follow that method instead.

```bash
sudo -E env \
  WIZ_ENABLE_DISK_SCANNER=1 \
  WIZ_SUBSCRIPTION_EXTERNAL_ID='dc-muc-vmware' \
  WIZ_SUBSCRIPTION_TAGS='{"Region":"dc-muc-vmware","vm_name":"dev-nvworkb01","owner":"kbe"}' \
  WIZ_API_CLIENT_ID="<YOUR_WIZ_CLIENT_ID>" \
  WIZ_API_CLIENT_SECRET="<YOUR_WIZ_CLIENT_SECRET>" \
  bash -c "$(curl -L [https://downloads.wiz.io/sensor/sensor_install.sh](https://downloads.wiz.io/sensor/sensor_install.sh))"
```

## 3. Verification & Service Diagnostics
Check the status of local components and verify successful data uploads to the Wiz backend:

### 3.1. Service Status
```bash
# Check if the Wiz Sensor daemon is active and running
sudo journalctl -u wiz-sensor -f --no-pager
sudo systemctl status wiz-sensor.service

# Inspect Ingestion Logs
sudo find /opt/ /var/log/ -iname "*wiz*" 2>/dev/null
sudo tail -n 50 /opt/wiz/sensor/host-store/sensor_logs/sensor.log

# Verify eBPF Kernel Probes
# Confirm active eBPF programs loaded into the Linux kernel by Wiz
sudo bpftool prog list | grep -i wiz
```
### 3.2. Test Runtime Sensor Detections
```bash
# 1. Activate the Hermes Agents Python environment
source /home/aiuser/hermes-agent/.venv/bin/activate

# 2. Trigger an agent execution that spawns sub-processes or network sockets
python -m hermes_agent --task "check-network-status"

# 3. Observe process execution logging captured by the Wiz Sensor
sudo journalctl -u wiz-sensor.service | grep -i "execve"
```
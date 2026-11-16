# Wiz Sensor Rollout with Ansible + Ansible Vault

Automated, repeatable deployment of the Wiz Runtime Sensor and Workload Scanner to on-prem Linux hosts – **without the Wiz service account secret ever appearing in a shell history, a process list, Ansible output or plain-text files in the repo.**

## How the secret is protected

| Risk | Protection |
|---|---|
| Secret in Git | Stored only in `group_vars/wiz_sensors/vault.yml`, encrypted with Ansible Vault (AES256) |
| Secret in shell history | Never typed on a command line – the vault password is prompted (`--ask-vault-pass`) |
| Secret in `ps` on the target host | Passed via a temporary root-only env file (`0600`), deleted right after the run. Ansible's `environment:` keyword is deliberately not used, because it puts the values on the remote command line |
| Secret in Ansible output / logs | `no_log: true` on all tasks that handle it |
| Secret on the host afterwards | Only in `/opt/wiz/sensor/host-store/engine.env` (root, `0400`) – required by the sensor, verified by the playbook |

## Structure

```text
ansible/
├── ansible.cfg
├── inventory.ini.example          # copy to inventory.ini (git-ignored)
├── wiz-sensor.yml                 # playbook
└── group_vars/wiz_sensors/
    ├── vars.yml                   # non-secret settings (subscription, tags, disk scanner, proxy)
    └── vault.yml.example          # template for the encrypted secrets
```

## Setup (once)

Requires `ansible-core` on the admin machine and SSH + sudo access to the target hosts.

```bash
cd ansible
cp inventory.ini.example inventory.ini            # add hosts
vi group_vars/wiz_sensors/vars.yml                # subscription, tags, proxy

# create the encrypted secrets file (opens an editor, saved encrypted)
cp group_vars/wiz_sensors/vault.yml.example group_vars/wiz_sensors/vault.yml
ansible-vault encrypt group_vars/wiz_sensors/vault.yml
ansible-vault edit    group_vars/wiz_sensors/vault.yml
```

Check before every commit that the file is encrypted:

```bash
head -1 group_vars/wiz_sensors/vault.yml          # must print: $ANSIBLE_VAULT;1.1;AES256
```

## Rollout

```bash
# Dry run: connectivity + which hosts would be installed
ansible wiz_sensors -m ping
ansible-playbook wiz-sensor.yml --ask-vault-pass --check

# Install (hosts with an existing sensor are skipped)
ansible-playbook wiz-sensor.yml --ask-vault-pass

# Single host only
ansible-playbook wiz-sensor.yml --ask-vault-pass --limit dev-nvworkb01
```

The playbook finally checks that `wiz-sensor.service` is running, that `engine.env` is root-only and which settings (subscription, tags, disk scanner) were written – names only, no values.

## Secret rotation

1. Create a new secret / service account in the Wiz console.
2. `ansible-vault edit group_vars/wiz_sensors/vault.yml` → replace the values.
3. Redeploy to all hosts:
   ```bash
   ansible-playbook wiz-sensor.yml --ask-vault-pass -e wiz_force_reinstall=true
   ```
4. Check the sensor status in the Wiz console, then delete the old secret / service account.

## Notes

- `--check` skips the installation itself (command tasks are not run in check mode).
- The installer is downloaded fresh on every run from `downloads.wiz.io`. Pin a version with `-e` / host_vars if needed (`WIZ_SENSOR_VERSION` – add it to the env block in the playbook).
- ⚠️ The expected format of `WIZ_SUBSCRIPTION_TAGS` (JSON is used here) should be checked against the Wiz on-prem documentation.
- Tested: syntax check, `ansible-lint` (production profile) and a local run against a mock installer – secret including special characters arrives intact, does not appear in `ps` or Ansible output, temporary files are removed. Not yet tested against the real Wiz installer.

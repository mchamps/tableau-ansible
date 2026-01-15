# Tableau Server Ansible Automation

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-2.10%2B-red.svg)](https://docs.ansible.com/)
[![Tableau](https://img.shields.io/badge/Tableau%20Server-2020.1%2B-orange.svg)](https://www.tableau.com/products/server)

Ansible playbooks and roles for automating Tableau Server administration, including backup, maintenance, and restore operations.

---

## Table of Contents

- [Features](#features)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Directory Structure](#directory-structure)
- [Playbooks](#playbooks)
- [Configuration](#configuration)
- [Scheduling](#scheduling)
- [Security](#security)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## Features

| Feature | Description |
|---------|-------------|
| **Automated Backups** | Schedule and execute TSM backups with timestamped filenames |
| **Log Management** | Archive and cleanup old log files automatically |
| **Maintenance Tasks** | Run TSM cleanup operations (logs, temp files, Redis cache) |
| **Settings Export** | Export ServerSettings.json for configuration tracking |
| **Email Reports** | Send HTML-formatted reports after backup completion |
| **Restore Operations** | Safely restore from backup files with validation |
| **Status Monitoring** | Check server status and list available backups |
| **Multi-Environment** | Separate configurations for production and development |

---

## How It Works

```
┌─────────────────┐      WinRM/HTTPS      ┌──────────────────────┐
│  Control Node   │ ──────────────────────▶│   Tableau Server     │
│  (Linux/macOS)  │                        │   (Windows)          │
│                 │                        │                      │
│  ┌───────────┐  │                        │  ┌────────────────┐  │
│  │ Ansible   │  │   1. TSM Login         │  │ TSM CLI        │  │
│  │ Playbook  │──│──────────────────────▶ │  │                │  │
│  └───────────┘  │   2. Run Maintenance   │  │ - backup       │  │
│                 │   3. Create Backup     │  │ - cleanup      │  │
│  ┌───────────┐  │   4. Cleanup Old Files │  │ - ziplogs      │  │
│  │ Vault     │  │   5. List Results      │  └────────────────┘  │
│  │ (secrets) │  │                        │                      │
│  └───────────┘  │                        │  ┌────────────────┐  │
│                 │ ◀──────────────────────│  │ Backup Files   │  │
│  ┌───────────┐  │   6. Return Status     │  │ *.tsbak        │  │
│  │ Send Email│  │                        │  └────────────────┘  │
│  │ Report    │  │                        │                      │
│  └───────────┘  │                        └──────────────────────┘
└─────────────────┘
```

**Backup Workflow:**
1. **Login** - Authenticate to TSM with encrypted credentials
2. **Export Settings** - Save ServerSettings.json for disaster recovery
3. **Archive Logs** - Run `tsm maintenance ziplogs` to compress logs
4. **Cleanup** - Run `tsm maintenance cleanup` to free disk space
5. **Backup** - Create timestamped `.tsbak` backup file
6. **Retention** - Delete backups older than retention period
7. **Report** - Send HTML email with operation results

---

## Prerequisites

### Control Node (where Ansible runs)

| Requirement | Version | Notes |
|-------------|---------|-------|
| Ansible | 2.10+ | `pip install ansible` |
| Python | 3.8+ | Required for Ansible |
| pywinrm | Latest | `pip install pywinrm` |

### Target Node (Tableau Server)

| Requirement | Version | Notes |
|-------------|---------|-------|
| Windows Server | 2016+ | With Tableau Server installed |
| Tableau Server | 2020.1+ | TSM-based installation |
| WinRM | Enabled | HTTPS on port 5986 |
| Firewall | Open | Port 5986 from control node |

### Install Ansible Collections

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Quick Start

### 1. Clone and Setup

```bash
git clone https://github.com/your-org/tableau-ansible.git
cd tableau-ansible
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Inventory

Edit `inventory/production/hosts`:

```ini
[tableau_servers]
tsm_primary ansible_host=tableau.example.com

[tableau_servers:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_winrm_server_cert_validation=ignore
ansible_port=5986
```

### 3. Configure Credentials (Ansible Vault)

```bash
# Copy the template
cp inventory/production/group_vars/vault.yml.example \
   inventory/production/group_vars/vault.yml

# Encrypt and edit
ansible-vault encrypt inventory/production/group_vars/vault.yml
ansible-vault edit inventory/production/group_vars/vault.yml
```

Add your credentials:

```yaml
# TSM Credentials
vault_tsm_username: "your_tsm_admin"
vault_tsm_password: "your_secure_password"

# Windows/WinRM Credentials
vault_ansible_user: "DOMAIN\\ansible_svc"
vault_ansible_password: "windows_password"

# Email Settings (optional)
vault_smtp_host: "smtp.example.com"
vault_email_from: "tableau-alerts@example.com"
```

### 4. Test Connectivity

```bash
ansible tableau_servers -m win_ping --ask-vault-pass
```

Expected output:
```
tsm_primary | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 5. Run Your First Backup

```bash
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass
```

---

## Directory Structure

```
tableau-ansible/
│
├── ansible.cfg                      # Ansible configuration
├── requirements.yml                 # Ansible Galaxy dependencies
├── LICENSE                          # MIT License
├── README.md                        # This file
│
├── inventory/
│   ├── production/
│   │   ├── hosts                    # Production servers
│   │   └── group_vars/
│   │       ├── tableau_servers.yml  # Production variables
│   │       └── vault.yml.example    # Credentials template
│   │
│   └── development/
│       ├── hosts                    # Development/test servers
│       └── group_vars/              # Development variables
│
├── playbooks/
│   ├── tableau_backup.yml           # Full backup + maintenance
│   ├── tableau_restore.yml          # Restore from backup
│   └── tableau_status.yml           # Health check
│
└── roles/
    ├── tableau_backup/              # Core backup operations
    │   ├── tasks/main.yml
    │   ├── defaults/main.yml
    │   └── handlers/main.yml
    │
    ├── tableau_cleanup/             # File retention management
    │   ├── tasks/main.yml
    │   └── defaults/main.yml
    │
    └── email_notification/          # Email reporting
        ├── tasks/main.yml
        ├── defaults/main.yml
        └── templates/
            └── email_report.html.j2
```

---

## Playbooks

### Backup (`tableau_backup.yml`)

Complete backup and maintenance workflow.

```bash
# Full backup (all tasks)
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass

# Backup only (skip cleanup and email)
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --tags "backup"

# Skip email notification
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --skip-tags "email"

# Dry run (see what would happen)
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --check

# Verbose output
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass -vvv
```

### Restore (`tableau_restore.yml`)

Restore Tableau Server from a backup file.

```bash
ansible-playbook playbooks/tableau_restore.yml --ask-vault-pass \
  -e "backup_file=ts_backup_20240115T120000.tsbak"
```

> **Warning**: The restore process will **stop Tableau Server**. Plan for downtime.

### Status (`tableau_status.yml`)

Quick health check and backup inventory.

```bash
ansible-playbook playbooks/tableau_status.yml --ask-vault-pass
```

**Output includes:**
- TSM service status
- Available backup files with sizes
- Disk space information

---

## Configuration

### Backup Settings

Edit `inventory/production/group_vars/tableau_servers.yml`:

```yaml
# Retention (how long to keep files)
backup_retention_days: 4        # Keep backups for 4 days
log_retention_days: 4           # Keep log archives for 4 days

# File naming
backup_filename_prefix: "ts_backup"
log_filename_prefix: "logs"

# Features
export_server_settings: true    # Export ServerSettings.json
```

### Maintenance Options

```yaml
maintenance_cleanup:
  logs: true           # -l: Clean up log files
  temp_files: true     # -t: Clean up temp files
  redis_cache: true    # -r: Clear Redis cache
  quiet_mode: true     # -q: Suppress prompts
```

### Email Notifications

```yaml
email_notification:
  enabled: true
  smtp_host: "{{ vault_smtp_host }}"
  smtp_port: 25
  smtp_use_tls: false
  from_address: "{{ vault_email_from }}"
  to_addresses:
    - "dba-team@example.com"
    - "oncall@example.com"
  subject_prefix: "TABLEAU BACKUP"
```

---

## Scheduling

### Cron (Linux/macOS)

```bash
# Edit crontab
crontab -e

# Daily backup at 2:00 AM
0 2 * * * cd /opt/tableau-ansible && \
  ansible-playbook playbooks/tableau_backup.yml \
  --vault-password-file=/etc/ansible/.vault_pass \
  >> /var/log/tableau-backup.log 2>&1

# Weekly full backup on Sundays at 1:00 AM
0 1 * * 0 cd /opt/tableau-ansible && \
  ansible-playbook playbooks/tableau_backup.yml \
  --vault-password-file=/etc/ansible/.vault_pass \
  -e "backup_retention_days=30" \
  >> /var/log/tableau-backup-weekly.log 2>&1
```

### Systemd Timer (Linux)

Create `/etc/systemd/system/tableau-backup.service`:
```ini
[Unit]
Description=Tableau Server Backup
After=network.target

[Service]
Type=oneshot
WorkingDirectory=/opt/tableau-ansible
ExecStart=/usr/bin/ansible-playbook playbooks/tableau_backup.yml --vault-password-file=/etc/ansible/.vault_pass
User=ansible

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/tableau-backup.timer`:
```ini
[Unit]
Description=Daily Tableau Backup

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:
```bash
systemctl enable --now tableau-backup.timer
```

### AWX / Ansible Tower

1. **Create Project** - Point to this Git repository
2. **Create Inventory** - Import from `inventory/production/hosts`
3. **Create Credentials** - Add Vault password and Windows credentials
4. **Create Job Template** - Select playbook and credentials
5. **Create Schedule** - Set desired frequency

---

## Security

### Best Practices Checklist

- [ ] **Use Ansible Vault** for all credentials
- [ ] **Never commit** `vault.yml` to version control
- [ ] **Separate credentials** per environment (prod/dev)
- [ ] **Limit WinRM access** via firewall rules
- [ ] **Use service accounts** with minimal permissions
- [ ] **Rotate credentials** periodically
- [ ] **Audit access** to the control node

### Vault Password File Security

```bash
# Create secure vault password file
echo 'your-vault-password' > ~/.vault_pass
chmod 600 ~/.vault_pass

# Use in playbooks
ansible-playbook playbooks/tableau_backup.yml \
  --vault-password-file=~/.vault_pass
```

### WinRM Security

Ensure WinRM is configured with HTTPS:

```powershell
# On Tableau Server (PowerShell as Admin)
winrm quickconfig -transport:https
winrm set winrm/config/service '@{AllowUnencrypted="false"}'
winrm set winrm/config/service/auth '@{Basic="false"}'
winrm set winrm/config/service/auth '@{Kerberos="true"}'
```

---

## Troubleshooting

### Connection Issues

| Problem | Solution |
|---------|----------|
| **Connection timeout** | Check firewall allows port 5986 |
| **Authentication failed** | Verify credentials in vault; check NTLM/Kerberos |
| **Certificate error** | Set `ansible_winrm_server_cert_validation=ignore` or install valid cert |
| **WinRM not listening** | Run `winrm quickconfig` on target |

**Test connectivity:**
```bash
# Basic ping test
ansible tableau_servers -m win_ping --ask-vault-pass

# Verbose connection test
ansible tableau_servers -m win_ping --ask-vault-pass -vvvv
```

### TSM Issues

| Problem | Solution |
|---------|----------|
| **TSM login failed** | Check TSM credentials; verify user has admin rights |
| **Backup path not found** | Run `tsm configuration get -k basefilepath.backuprestore` |
| **Insufficient disk space** | Check disk space; adjust retention settings |
| **Backup timeout** | Increase `timeout` in ansible.cfg |

**Manual TSM verification:**
```powershell
# On Tableau Server
tsm status -v
tsm configuration get -k basefilepath.backuprestore
tsm configuration get -k basefilepath.log_archive
```

### Debug Mode

```bash
# Maximum verbosity
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass -vvvv

# Step through tasks one at a time
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --step

# Start at a specific task
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass \
  --start-at-task="Run TSM maintenance backup"
```

---

## FAQ

**Q: How long do backups take?**
A: Depends on data size. Expect 10-60 minutes for typical deployments. The server remains online during backup.

**Q: Can I run backups during business hours?**
A: Yes, but expect some performance impact. Off-hours is recommended for large instances.

**Q: How much disk space do backups need?**
A: Each `.tsbak` file is typically 50-80% of your Tableau repository size. Plan for `(retention_days + 1) * backup_size`.

**Q: Can I backup to a network share?**
A: Yes, configure `basefilepath.backuprestore` in TSM to point to a UNC path. Ensure the TSM service account has write access.

**Q: How do I restore to a different server?**
A: Copy the `.tsbak` file to the new server's backup directory, then run the restore playbook against the new server.

**Q: What Tableau Server versions are supported?**
A: Any TSM-based installation (2018.2+). Tested primarily on 2020.1 through 2023.3.

---

## Contributing

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/amazing-feature`)
3. **Test** your changes in a development environment
4. **Commit** with clear messages (`git commit -m 'Add amazing feature'`)
5. **Push** to your branch (`git push origin feature/amazing-feature`)
6. **Open** a Pull Request

### Development Setup

```bash
# Clone your fork
git clone https://github.com/YOUR_USERNAME/tableau-ansible.git

# Use development inventory
ansible-playbook playbooks/tableau_backup.yml \
  -i inventory/development/hosts \
  --ask-vault-pass
```

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Resources

- [Tableau Server TSM Command Reference](https://help.tableau.com/current/server/en-us/tsm.htm)
- [Ansible Windows Documentation](https://docs.ansible.com/ansible/latest/os_guide/windows_usage.html)
- [Ansible Vault Documentation](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

---

<p align="center">
  <i>Built with Ansible for Tableau Server administrators</i>
</p>

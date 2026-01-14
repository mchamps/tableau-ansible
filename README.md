# Tableau Server Ansible Automation

Ansible playbooks and roles for automating Tableau Server administration, including backup, maintenance, and restore operations.

## Features

- **Automated Backups**: Schedule and execute TSM backups with timestamped filenames
- **Log Management**: Archive and cleanup old log files automatically
- **Maintenance Tasks**: Run TSM cleanup operations (logs, temp files, Redis cache)
- **Server Settings Export**: Export ServerSettings.json for configuration tracking
- **Email Notifications**: Send HTML-formatted reports after backup completion
- **Restore Operations**: Safely restore from backup files with validation
- **Status Monitoring**: Check server status and list available backups

## Prerequisites

- **Ansible**: Version 2.10 or higher
- **Python**: 3.8 or higher
- **Target System**: Windows Server with Tableau Server installed
- **Network**: WinRM connectivity to Tableau Server (port 5986)
- **Credentials**: TSM administrator credentials

### Required Ansible Collections

Install required collections before running playbooks:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/tableau-ansible.git
cd tableau-ansible
```

### 2. Install Dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

### 3. Configure Inventory

Edit `inventory/production/hosts` with your Tableau Server details:

```ini
[tableau_servers]
tsm_primary ansible_host=your-tableau-server.example.com
```

### 4. Configure Credentials

Create a vault file for sensitive credentials:

```bash
cp inventory/production/group_vars/vault.yml.example inventory/production/group_vars/vault.yml
ansible-vault encrypt inventory/production/group_vars/vault.yml
ansible-vault edit inventory/production/group_vars/vault.yml
```

Add your credentials:

```yaml
vault_tsm_username: "your_tsm_admin"
vault_tsm_password: "your_password"
vault_ansible_user: "windows_user"
vault_ansible_password: "windows_password"
vault_smtp_host: "smtp.example.com"
vault_email_from: "tableau@example.com"
```

### 5. Run a Backup

```bash
# With vault password prompt
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass

# Or with vault password file
ansible-playbook playbooks/tableau_backup.yml --vault-password-file=~/.vault_pass
```

## Directory Structure

```
tableau-ansible/
├── ansible.cfg                 # Ansible configuration
├── requirements.yml            # Ansible Galaxy dependencies
├── inventory/
│   ├── production/
│   │   ├── hosts              # Production inventory
│   │   └── group_vars/
│   │       ├── tableau_servers.yml  # Server variables
│   │       └── vault.yml.example    # Vault template
│   └── development/
│       └── hosts              # Development inventory
├── playbooks/
│   ├── tableau_backup.yml     # Main backup playbook
│   ├── tableau_restore.yml    # Restore playbook
│   └── tableau_status.yml     # Status check playbook
└── roles/
    ├── tableau_backup/        # Backup and maintenance role
    ├── tableau_cleanup/       # File cleanup role
    └── email_notification/    # Email reporting role
```

## Available Playbooks

### Backup (`tableau_backup.yml`)

Performs complete backup and maintenance cycle:

```bash
# Full backup with all tasks
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass

# Skip email notification
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --skip-tags "email"

# Only run backup (skip cleanup)
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --tags "backup"

# Dry run (check mode)
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass --check
```

### Restore (`tableau_restore.yml`)

Restores from a backup file:

```bash
ansible-playbook playbooks/tableau_restore.yml --ask-vault-pass \
  -e "backup_file=ts_backup_20240115T120000.tsbak"
```

**Warning**: This stops the server during restore!

### Status (`tableau_status.yml`)

Checks server status and lists backups:

```bash
ansible-playbook playbooks/tableau_status.yml --ask-vault-pass
```

## Configuration Options

### Backup Settings (`group_vars/tableau_servers.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `backup_retention_days` | 4 | Days to keep backup files |
| `log_retention_days` | 4 | Days to keep log archives |
| `backup_filename_prefix` | ts_backup | Prefix for backup files |
| `export_server_settings` | true | Export ServerSettings.json |

### Maintenance Options

| Variable | Default | Description |
|----------|---------|-------------|
| `maintenance_cleanup.logs` | true | Clean up log files (-l) |
| `maintenance_cleanup.temp_files` | true | Clean up temp files (-t) |
| `maintenance_cleanup.redis_cache` | true | Clear Redis cache (-r) |
| `maintenance_cleanup.quiet_mode` | true | Quiet mode (-q) |

### Email Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `email_notification.enabled` | true | Enable email notifications |
| `email_notification.smtp_host` | - | SMTP server hostname |
| `email_notification.smtp_port` | 25 | SMTP port |
| `email_notification.smtp_use_tls` | false | Use TLS encryption |
| `email_notification.to_addresses` | [] | List of recipients |

## Security Best Practices

1. **Always use Ansible Vault** for credentials
2. **Never commit** `vault.yml` or plain-text passwords
3. **Use separate credentials** for development and production
4. **Limit WinRM access** to Ansible control nodes only
5. **Review backup retention** policies regularly

## Scheduling Backups

### Using Cron (Linux Control Node)

```bash
# Daily backup at 2 AM
0 2 * * * cd /path/to/tableau-ansible && ansible-playbook playbooks/tableau_backup.yml --vault-password-file=~/.vault_pass >> /var/log/tableau-backup.log 2>&1
```

### Using AWX/Ansible Tower

1. Create a Project pointing to this repository
2. Create Job Templates for each playbook
3. Add Vault credentials
4. Create Schedules as needed

## Troubleshooting

### Common Issues

**WinRM Connection Failed**
```bash
# Test connectivity
ansible tableau_servers -m win_ping --ask-vault-pass
```

**TSM Login Failed**
- Verify TSM credentials in vault
- Ensure user has TSM administrator permissions

**Backup Folder Not Found**
- TSM may not be properly initialized
- Run `tsm configuration get -k basefilepath.backuprestore` manually

### Debug Mode

```bash
ansible-playbook playbooks/tableau_backup.yml --ask-vault-pass -vvv
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test in a development environment
5. Submit a pull request

## License

MIT License - See LICENSE file for details.

## Support

For issues and feature requests, please open an issue on GitHub.

# Ansible role for Vykar

This role will setup [Vykar](https://vykar.borgbase.com/) backups on a Linux machine using a long-running systemd service (`vykar daemon`). It focuses on simplicity and avoids adding another layer of complexity by using vykar's built-in YAML configuration and scheduling.

Vykar is a fast, encrypted, deduplicated backup tool written in Rust by BorgBase ([GitHub](https://github.com/borgbase/vykar)). It supports local, S3, SFTP, and REST server backends.

## Role Variables

### Vykar installation

The role will download and install the vykar binary (version `vykar_version`) into `vykar_path`. On each run, it checks the installed version and upgrades if a newer version is available.

- `vykar_version`: Version of vykar to install (default: `"latest"`)
- `vykar_path`: Installation path for the binary (default: `/usr/local/bin/vykar`)

Both `x86_64` and `aarch64` architectures are supported (musl builds).

### Vykar configuration

- `vykar_config_dir`: Directory for vykar configuration (default: `/etc/vykar`)
- `vykar_config`: **Required**. A YAML dict with the full vykar configuration. Rendered via `to_nice_yaml` into `/etc/vykar/config.yaml`.

See the [vykar configuration documentation](https://vykar.borgbase.com/configuration) for all available options and the [backends documentation](https://vykar.borgbase.com/backends) for supported storage backends.

### Repository initialization

The role automatically handles repository initialization:
- Checks if the repository exists by running `vykar info`
- If the repository doesn't exist, runs `vykar init` automatically

### Systemd service

A `vykar-daemon.service` service will be created that runs `vykar daemon`. The daemon is long-running and handles its own backup scheduling based on the `schedule` section in the config file.

You can see the logs with `journalctl`:

```bash
journalctl -xefu vykar-daemon
```

## Example playbook

### Basic example with local repository

```yaml
---
- hosts: myserver
  roles:
    - role: vykar
      vars:
        vykar_config:
          repositories:
            - url: "/backup/repo"
          encryption:
            passphrase: "mysuperduperpassword"
          sources:
            - "/home"
            - "/etc"
          retention:
            keep_daily: 7
          schedule:
            every: "24h"
```

### Example with S3 backend

```yaml
---
- hosts: myserver
  roles:
    - role: vykar
      vars:
        vykar_config:
          repositories:
            - url: "s3://s3.us-east-1.amazonaws.com/my-backup-bucket/repo"
              access_key_id: "XXXXX"
              secret_access_key: "XXXXX"
          encryption:
            passphrase: "mysuperduperpassword"
          sources:
            - "/home"
            - "/var/www"
            - "/etc"
          retention:
            keep_daily: 7
            keep_weekly: 4
          schedule:
            cron: "0 3 * * *"  # daily at 3:00 AM
```

### Example with database dumps using command_dumps

```yaml
---
- hosts: myserver
  roles:
    - role: vykar
      vars:
        vykar_config:
          repositories:
            - url: "/backup/repo"
          encryption:
            passphrase: "mysuperduperpassword"
          sources:
            - label: "databases"
              command_dumps:
                - name: mydb.sql
                  command: "sudo -u postgres pg_dump mydb"
            - "/etc"
          retention:
            keep_daily: 7
          schedule:
            every: "24h"
```

## Managing backups

After the role is applied, you can interact with vykar directly:

```bash
# View snapshots
vykar snapshots

# View repository info
vykar info

# Check daemon status
systemctl status vykar-daemon

# Run manual backup
vykar backup
```

## Security Considerations

### File Permissions
- Config directory (`/etc/vykar`): `0700` (only root can access)
- Config file (`/etc/vykar/config.yaml`): `0600` (only root can read/write)
- Binary (`/usr/local/bin/vykar`): `0755` (executable by all, writable only by root)

The config file contains sensitive data (passphrases, cloud credentials) and is protected with restrictive permissions.

### Systemd Service Hardening
The systemd service includes security hardening features:
- `ProtectSystem=strict`: Makes system directories read-only
- `ProtectHome=read-only`: Protects home directories (allows read for backup)
- `PrivateTmp=true`: Provides isolated /tmp directory
- `NoNewPrivileges=true`: Prevents privilege escalation
- `PrivateDevices=true`: Restricts device access

### Best Practices
- **Use ansible-vault**: Encrypt `vykar_config` or at least the passphrase to protect credentials
- **Regular updates**: Using `vykar_version: "latest"` (the default) ensures you get updates on each role run

## License

MIT

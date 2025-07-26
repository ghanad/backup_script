# Custom Backup Script with Prometheus Metrics

A robust Bash script designed to back up specified files and directories, transfer them to one or more remote servers via `rsync`, and generate detailed metrics for monitoring with Prometheus and Grafana.

## Features

-   **Multi-Destination Backups**: Easily configure multiple remote SFTP/SSH servers to store backup archives.
-   **Prometheus Integration**: Generates detailed metrics for `node_exporter`'s textfile collector.
-   **Comprehensive Metrics**: Tracks job success, duration, archive size, last success timestamp, and per-destination transfer status.
-   **Dynamic Naming**: Automatically names backup files using the server's IP address and the current date.
-   **File Exclusion**: Easily specify patterns to exclude certain files or directories (e.g., `*.log`) from the backup.
-   **Automated Cleanup**: Automatically deletes old backups on remote servers after a configurable number of days.
-   **Robust & Secure**: Designed for automation via cron, enforces the use of SSH keys, and includes self-healing permission logic.
-   **Versioning**: Includes a script version that is exposed as a Prometheus metric for easy tracking of deployments.

## Prerequisites

Before using this script, ensure the following are installed on the server:
-   `bash` (v4.0 or later)
-   `rsync`
-   `ssh` client
-   `tar`, `gzip`, `grep`, `awk`, `stat`, `pidof`
-   `node_exporter` service running and configured to read from the textfile collector directory.

## Configuration

All configuration is done via variables at the top of the `backup.sh` script.

| Variable                 | Description                                                                                                                             | Example                                                                  |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| `SCRIPT_VERSION`         | The version of the script. Exposed as a metric.                                                                                         | `"1.0.0"`                                                                |
| `SOURCE_ITEMS`           | An array of absolute paths to files and directories to back up.                                                                         | `("/etc" "/var/lib/grafana/")`                                            |
| `EXCLUDE_PATTERNS`       | An array of patterns (passed to `tar --exclude`) to exclude from the archive.                                                           | `("*.log" "*.tmp")`                                                       |
| `DESTINATIONS`           | An array of remote server connection strings. **Format**: `"user@server:port:/base/path"`                                                 | `("dcadmin@192.168.10.10:22:/home/dcadmin/backups")`                      |
| `PRIVATE_KEY_PATH`       | **(Mandatory)** Absolute path to the SSH private key for passwordless authentication.                                                     | `"/root/.ssh/id_rsa_backup"`                                             |
| `DAYS_TO_KEEP_REMOTE`    | Number of days to keep old backups on remote servers. Set to `0` to disable cleanup.                                                    | `30`                                                                     |
| `TEXTFILE_COLLECTOR_DIR` | The directory where `node_exporter` reads metrics from.                                                                                 | `"/var/lib/node_exporter/textfile_collector"`                            |

## Installation & Setup

Follow these steps to deploy and configure the script on a new server.

### Step 1: Copy and Make Executable

1.  Copy the `backup.sh` script to a suitable location (e.g., `/usr/local/bin/`).
2.  Make it executable:
    ```bash
    chmod +x /usr/local/bin/backup.sh
    ```

### Step 2: Configure SSH Key (Mandatory for Automation)

This script is designed for automated execution and requires a passwordless SSH key.

1.  **Generate a new key** (recommended) on the source server:
    ```bash
    ssh-keygen -t rsa -b 4096 -f /root/.ssh/backup_key -N ""
    ```
2.  **Copy the public key** to each destination server:
    ```bash
    ssh-copy-id -i /root/.ssh/backup_key.pub dcadmin@192.168.10.10
    ```
3.  Update the `PRIVATE_KEY_PATH` variable in the script to point to `/root/.ssh/backup_key`.

### Step 3: Configure Directory Permissions (Mandatory One-Time Setup)

> **IMPORTANT:** This is the most critical setup step. The `node_exporter` service runs as a non-root user (e.g., `node_exporter`) and needs permission to access its textfile collector directory. The following commands prepare the environment securely. This prevents `permission denied` errors in `node_exporter` logs.

These commands should be run **once** on each server where the script is deployed.

```bash
# Find the group node_exporter runs as (usually 'node_exporter')
# You can confirm with: id node_exporter

# 1. Set group ownership on the parent directory
sudo chown root:node_exporter /var/lib/node_exporter/

# 2. Set permissions for the parent directory (rwxr-x---)
# Allows the node_exporter group to enter this directory.
sudo chmod 750 /var/lib/node_exporter/

# 3. Set group ownership on the textfile collector directory
sudo chown root:node_exporter /var/lib/node_exporter/textfile_collector/

# 4. Set permissions for the textfile collector directory (rwxrwx---)
# Allows the script (as root) and node_exporter group members to manage files.
sudo chmod 770 /var/lib/node_exporter/textfile_collector/
```

# Generated Metrics

The script generates a `custom_backup.prom` file with the following metrics:

| Metric Name | Description | Labels |
|-------------|-------------|--------|
| `custom_backup_info` | Static info about the script. The value is always 1. | `server_ip`, `version` |
| `custom_backup_job_success` | Overall job status (1 for success, 0 for failure). | `server_ip` |
| `custom_backup_job_duration_seconds` | Total job execution time in seconds. | `server_ip` |
| `custom_backup_archive_size_bytes` | The size of the final .tar.gz archive in bytes. | `server_ip` |
| `custom_backup_last_success_timestamp_seconds` | The Unix timestamp of the last fully successful job run. | `server_ip` |
| `custom_backup_transfer_success` | Transfer status to a specific remote server (1 or 0). | `server_ip`, `destination` |
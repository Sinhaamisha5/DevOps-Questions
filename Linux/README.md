# Bash Scripts Collection

A collection of practical bash scripts for system administration, monitoring, and automation tasks.

## 📋 Table of Contents

- [Scripts Overview](#scripts-overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Script Details](#script-details)
  - [1. System Resource Monitor](#1-system-resource-monitor)
  - [2. File Cleanup Script](#2-file-cleanup-script)
  - [3. Log File Analyzer](#3-log-file-analyzer)
  - [4. User Management Script](#4-user-management-script)
  - [5. Configuration Backup](#5-configuration-backup)
  - [6. Interactive User Menu](#6-interactive-user-menu)
  - [7. Weather Information Fetcher](#7-weather-information-fetcher)
  - [8. IP Address Validator](#8-ip-address-validator)
  - [9. Service Monitor](#9-service-monitor)
  - [10. Server File Sync](#10-server-file-sync)
- [License](#license)

## Scripts Overview

| Script | Purpose | Scheduling |
|--------|---------|------------|
| System Monitor | Logs CPU/memory usage | Every 5 minutes (cron) |
| File Cleanup | Deletes old files (30+ days) | As needed or scheduled |
| Log Analyzer | Finds top 10 requested URLs | On-demand |
| User Manager | Creates users if they don't exist | On-demand |
| Config Backup | Backs up /etc with timestamps | Daily recommended |
| User Menu | Interactive user management | On-demand |
| Weather Fetcher | Gets weather via API | On-demand |
| IP Validator | Validates IP addresses | On-demand |
| Service Monitor | Monitors and restarts services | Every 5-10 minutes |
| File Sync | Syncs files between servers | Scheduled |

## Prerequisites

- Linux/Unix operating system
- Bash 4.0 or higher
- Root/sudo access (for some scripts)
- Required commands: `top`, `vmstat`, `free`, `awk`, `curl`

## Installation

1. Clone this repository:
```bash
git clone https://github.com/yourusername/bash-scripts.git
cd bash-scripts
```

2. Make scripts executable:
```bash
chmod +x *.sh
```

3. (Optional) Copy scripts to a directory in your PATH:
```bash
sudo cp *.sh /usr/local/bin/
```

## Script Details

### 1. System Resource Monitor

Monitors CPU and memory usage, logging output every 5 minutes.

**File:** `system_monitor.sh`

```bash
#!/bin/bash
date >> /var/log/sys_usage.log
top -b -n1 | head -5 >> /var/log/sys_usage.log
free -h >> /var/log/sys_usage.log
```

**How it works:**
- `date` - Records timestamp for each log entry
- `top -b -n1` - Captures CPU usage in batch mode (single iteration)
- `head -5` - Takes first 5 lines showing CPU load and top processes
- `free -h` - Displays memory usage in human-readable format
- `>>` - Appends output to log file

**Setup cron job:**
```bash
# Edit crontab
crontab -e

# Add this line to run every 5 minutes
*/5 * * * * /path/to/system_monitor.sh
```

**View logs:**
```bash
tail -f /var/log/sys_usage.log
```

---

### 2. File Cleanup Script

Automatically finds and deletes files older than 30 days in a specified directory.

**File:** `cleanup_old_files.sh`

```bash
#!/bin/bash

TARGET_DIR="/path/to/directory"

find "$TARGET_DIR" -type f -mtime +30 -exec rm -f {} \;

echo "$(date): Deleted files older than 30 days in $TARGET_DIR" >> /var/log/cleanup.log
```

**How it works:**
- `find "$TARGET_DIR"` - Searches in the target directory
- `-type f` - Only matches files (not directories)
- `-mtime +30` - Finds files modified more than 30 days ago
- `-exec rm -f {} \;` - Deletes each file found
- Logs cleanup action with timestamp

**Usage:**
```bash
# Edit TARGET_DIR in the script first
./cleanup_old_files.sh
```

**⚠️ Warning:** Test with `-print` instead of `-exec rm` first:
```bash
find "$TARGET_DIR" -type f -mtime +30 -print
```

---

### 3. Log File Analyzer

Parses Apache/Nginx access logs to find the top 10 most requested URLs.

**File:** `log_analyzer.sh`

```bash
#!/bin/bash

LOGFILE="/var/log/nginx/access.log"

echo "Top 10 requested URLs:"
awk '{print $7}' "$LOGFILE" | sort | uniq -c | sort -nr | head -10
```

**How it works:**
- `awk '{print $7}'` - Extracts the 7th field (URL path) from each log line
- `sort` - Sorts all URLs alphabetically
- `uniq -c` - Counts occurrences of each unique URL
- `sort -nr` - Sorts numerically in reverse order (highest first)
- `head -10` - Displays top 10 results

**Usage:**
```bash
# For Nginx logs
./log_analyzer.sh

# For Apache logs (usually same format)
LOGFILE="/var/log/apache2/access.log" ./log_analyzer.sh
```

**Sample output:**
```
Top 10 requested URLs:
   1523 /index.html
    847 /api/users
    632 /images/logo.png
    ...
```

---

### 4. User Management Script

Checks if a user exists and creates it if not found.

**File:** `user_check_create.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Accept username as parameter
- Check if user exists using `id` or `/etc/passwd`
- Create user with `useradd` if not found
- Set default shell and home directory
- Log user creation

---

### 5. Configuration Backup

Automates backup of `/etc/` directory with date-based filenames.

**File:** `backup_etc.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Create compressed backup of `/etc/`
- Use date-based naming: `etc_backup_YYYY-MM-DD.tar.gz`
- Store in designated backup directory
- Optional rotation to keep only last N backups
- Verification of backup integrity

---

### 6. Interactive User Menu

Menu-driven script for user management operations.

**File:** `user_menu.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Add new user
- Delete existing user
- List all users
- Modify user properties
- Interactive menu interface

---

### 7. Weather Information Fetcher

Fetches current weather information using a weather API.

**File:** `weather_fetch.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Use `curl` to fetch from weather API (e.g., OpenWeatherMap)
- Parse JSON response
- Display temperature, conditions, humidity
- Accept location as parameter
- Format output in readable format

---

### 8. IP Address Validator

Validates IPv4 addresses using regular expressions.

**File:** `ip_validator.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Validate IPv4 format using regex
- Check for valid octet ranges (0-255)
- Accept IP as parameter or from stdin
- Return exit code for use in other scripts

---

### 9. Service Monitor

Monitors critical services and restarts them if down.

**File:** `service_monitor.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Check status of services (ssh, docker, nginx, etc.)
- Restart service if not running
- Send alerts or log warnings
- Support multiple services in one run
- Configurable service list

---

### 10. Server File Sync

Cron job script to sync files between two servers.

**File:** `server_sync.sh`

**Status:** 🚧 To be implemented

**Planned features:**
- Use `rsync` for efficient file synchronization
- SSH key-based authentication
- Bidirectional or unidirectional sync
- Exclude patterns for sensitive files
- Logging and error handling

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - feel free to use these scripts in your own projects.

---

**Note:** Always test scripts in a safe environment before running in production. Some scripts require root privileges and can modify system files.
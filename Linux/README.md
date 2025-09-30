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
- Required commands: `top`, `vmstat`, `free`, `awk`, `curl`, `jq`, `rsync`
- For Weather Fetcher: OpenWeatherMap API key (free at [openweathermap.org](https://openweathermap.org/api))

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

Checks if users exist and creates them if not found, reading from a file.

**File:** `user_check_create.sh`

**Input file format (`users.txt`):**
```
alice:Alice123
bob:Bob456
charlie:Charlie789
```

**Script:**
```bash
#!/bin/bash

FILE="users.txt"

while IFS=: read -r username password; do
    # Skip empty lines
    [[ -z "$username" || -z "$password" ]] && continue

    # Check if user exists
    if id "$username" &>/dev/null; then
        echo "User '$username' already exists."
    else
        # Create user with home directory
        sudo useradd -m "$username"

        # Set password
        echo "$username:$password" | sudo chpasswd

        echo "User '$username' created successfully."
    fi
done < "$FILE"
```

**How it works:**
- `while IFS=: read -r username password` - Reads each line from the file, splitting by `:`
- `[[ -z "$username" || -z "$password" ]] && continue` - Skips empty or malformed lines
- `id "$username" &>/dev/null` - Checks if the user exists
- `useradd -m "$username"` - Creates user with a home directory
- `echo "$username:$password" | sudo chpasswd` - Sets the password
- `done < "$FILE"` - Loops through all lines in the file

**Usage:**
```bash
# Create users.txt with username:password format
./user_check_create.sh
```

---

### 5. Configuration Backup

Automates backup of `/etc/` directory with date-based filenames.

**File:** `backup_etc.sh`

```bash
#!/bin/bash

# Directory to back up
SOURCE_DIR="/etc"

# Backup destination
BACKUP_DIR="/backup"

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Generate date-based filename
DATE=$(date +%F)  # Format: YYYY-MM-DD
BACKUP_FILE="$BACKUP_DIR/etc_backup_$DATE.tar.gz"

# Create the backup
tar -czf "$BACKUP_FILE" "$SOURCE_DIR"

# Optional: log the backup
echo "$(date): Backup of $SOURCE_DIR completed at $BACKUP_FILE" >> /var/log/backup.log
```

**How it works:**
- `SOURCE_DIR="/etc"` - Directory to back up
- `BACKUP_DIR="/backup"` - Destination directory for backups
- `mkdir -p "$BACKUP_DIR"` - Ensures the backup directory exists
- `DATE=$(date +%F)` - Gets current date in YYYY-MM-DD format
- `tar -czf "$BACKUP_FILE" "$SOURCE_DIR"` - Creates a compressed archive (.tar.gz)
- Optional logging appends backup info to `/var/log/backup.log`

**Setup cron job:**
```bash
# Run daily at 2 AM
0 2 * * * /path/to/backup_etc.sh
```

**Enhanced version with retention (keeps last 7 backups):**

```bash
#!/bin/bash

SOURCE_DIR="/etc"
BACKUP_DIR="/backup"

mkdir -p "$BACKUP_DIR"

DATE=$(date +%F)
BACKUP_FILE="$BACKUP_DIR/etc_backup_$DATE.tar.gz"

# Create the backup
tar -czf "$BACKUP_FILE" "$SOURCE_DIR"
echo "$(date): Backup of $SOURCE_DIR completed at $BACKUP_FILE" >> /var/log/backup.log

# Keep only the last 7 backups
cd "$BACKUP_DIR" || exit
BACKUPS_TO_DELETE=$(ls -1t etc_backup_*.tar.gz | tail -n +8)
if [ -n "$BACKUPS_TO_DELETE" ]; then
    echo "$BACKUPS_TO_DELETE" | xargs rm -f
    echo "$(date): Deleted old backups:" >> /var/log/backup.log
    echo "$BACKUPS_TO_DELETE" >> /var/log/backup.log
fi
```

**Retention explanation:**
- `ls -1t etc_backup_*.tar.gz` - Lists backups in reverse chronological order (newest first)
- `tail -n +8` - Skips the first 7 backups, listing the older ones to delete
- `xargs rm -f` - Deletes those older backups

---

### 6. Interactive User Menu

Menu-driven script for user management operations.

**File:** `user_menu.sh`

```bash
#!/bin/bash

while true; do
    echo "============================="
    echo " User Management Menu "
    echo "============================="
    echo "1. Add User"
    echo "2. Delete User"
    echo "3. List Users"
    echo "4. Exit"
    echo "============================="
    read -p "Enter your choice [1-4]: " choice

    case $choice in
        1)
            read -p "Enter username to add: " username
            if id "$username" &>/dev/null; then
                echo "User '$username' already exists."
            else
                sudo useradd -m "$username"
                echo "User '$username' added successfully."
            fi
            ;;
        2)
            read -p "Enter username to delete: " username
            if id "$username" &>/dev/null; then
                sudo userdel -r "$username"
                echo "User '$username' deleted successfully."
            else
                echo "User '$username' does not exist."
            fi
            ;;
        3)
            echo "Current system users:"
            cut -d: -f1 /etc/passwd
            ;;
        4)
            echo "Exiting..."
            exit 0
            ;;
        *)
            echo "Invalid option. Please choose 1-4."
            ;;
    esac

    echo ""
done
```

**How it works:**
- `while true; do ... done` - Keeps showing the menu until user exits
- `read -p` - Reads user input
- `case $choice in` - Handles different menu options
- **Add User:** Checks if user exists with `id`, creates with `useradd -m`
- **Delete User:** Removes user and home directory with `userdel -r`
- **List Users:** Extracts usernames from `/etc/passwd` using `cut`
- **Exit:** Exits the script gracefully

**Usage:**
```bash
./user_menu.sh
```

---

### 7. Weather Information Fetcher

Fetches current weather information using OpenWeatherMap API.

**File:** `weather_fetch.sh`

```bash
#!/bin/bash

# Replace with your OpenWeatherMap API key
API_KEY="YOUR_API_KEY"

# Ask for city
read -p "Enter city name: " CITY

# Fetch weather data in JSON format
RESPONSE=$(curl -s "http://api.openweathermap.org/data/2.5/weather?q=${CITY}&appid=${API_KEY}&units=metric")

# Parse JSON using jq (recommended)
TEMP=$(echo "$RESPONSE" | jq '.main.temp')
DESC=$(echo "$RESPONSE" | jq -r '.weather[0].description')
HUMIDITY=$(echo "$RESPONSE" | jq '.main.humidity')
WIND=$(echo "$RESPONSE" | jq '.wind.speed')

# Display the result
echo "Weather in $CITY:"
echo "Temperature: $TEMP °C"
echo "Description: $DESC"
echo "Humidity: $HUMIDITY %"
echo "Wind Speed: $WIND m/s"
```

**How it works:**
- `curl -s` - Fetches data silently (no progress info)
- **API URL:** `http://api.openweathermap.org/data/2.5/weather?q=<city>&appid=<API_KEY>&units=metric`
  - `q=<city>` - City name
  - `appid=<API_KEY>` - Your API key
  - `units=metric` - Temperature in Celsius
- `jq` - Parses JSON output:
  - `.main.temp` - Temperature
  - `.weather[0].description` - Weather description
  - `.main.humidity` - Humidity percentage
  - `.wind.speed` - Wind speed

**Prerequisites:**
- Install `jq`: `sudo apt install jq` (Debian/Ubuntu) or `sudo yum install jq` (RHEL/CentOS)
- Get free API key from [openweathermap.org](https://openweathermap.org/api)

**Usage:**
```bash
./weather_fetch.sh
# Enter city name when prompted
```

---

### 8. IP Address Validator

Validates IPv4 addresses using regular expressions.

**File:** `ip_validator.sh`

```bash
#!/bin/bash

read -p "Enter an IP address: " ip

# Regular expression for IPv4 validation
regex="^([0-9]{1,3}\.){3}[0-9]{1,3}$"

if [[ $ip =~ $regex ]]; then
    # Further check each octet is <= 255
    valid=true
    IFS='.' read -r -a octets <<< "$ip"
    for octet in "${octets[@]}"; do
        if (( octet < 0 || octet > 255 )); then
            valid=false
            break
        fi
    done

    if $valid; then
        echo "Valid IP address."
    else
        echo "Invalid IP address: octet out of range (0-255)."
    fi
else
    echo "Invalid IP address: format incorrect."
fi
```

**How it works:**
- `regex="^([0-9]{1,3}\.){3}[0-9]{1,3}$"` - Matches 4 groups of 1-3 digits separated by dots
- `[[ $ip =~ $regex ]]` - Checks if input matches the regex pattern
- **Further validation:**
  - Splits the IP into octets using `IFS='.'`
  - Checks each octet is between 0 and 255
- Provides clear messages for valid/invalid IPs

**Usage:**
```bash
./ip_validator.sh
# Enter IP address when prompted

# Examples:
# Valid: 192.168.1.1
# Invalid: 256.100.50.25 (256 is out of range)
# Invalid: 192.168.1 (incomplete format)
```

---

### 9. Service Monitor

Monitors critical services and restarts them if down.

**File:** `service_monitor.sh`

```bash
#!/bin/bash

# List of services to monitor
SERVICES=("ssh" "docker")

for service in "${SERVICES[@]}"; do
    # Check if the service is active
    if systemctl is-active --quiet "$service"; then
        echo "$(date): $service is running."
    else
        echo "$(date): $service is down. Restarting..."
        sudo systemctl restart "$service"

        # Verify restart
        if systemctl is-active --quiet "$service"; then
            echo "$(date): $service restarted successfully."
        else
            echo "$(date): Failed to restart $service."
        fi
    fi
done
```

**How it works:**
- `SERVICES=("ssh" "docker")` - Array of services to monitor
- `systemctl is-active --quiet "$service"` - Checks if the service is running
- **If down:**
  - `sudo systemctl restart "$service"` - Restarts the service
  - Verifies restart succeeded
- Logging with `$(date)` tracks when services went down and were restarted

**Setup cron job:**
```bash
# Run every 5 minutes
*/5 * * * * /path/to/service_monitor.sh
```

**Usage:**
```bash
# Edit SERVICES array to monitor your services
./service_monitor.sh

# Or run with logging
./service_monitor.sh >> /var/log/service_monitor.log 2>&1
```

---

### 10. Server File Sync

Cron job script to sync files between two servers using rsync.

**File:** `server_sync.sh`

```bash
#!/bin/bash

# Source directory (local)
SRC_DIR="/path/to/source/"

# Destination (remote)
DEST_USER="remoteuser"
DEST_HOST="remote.server.com"
DEST_DIR="/path/to/destination/"

# Rsync options:
# -a → archive mode (preserves permissions, timestamps, etc.)
# -v → verbose
# -z → compress during transfer
# -h → human-readable
# --delete → delete files at destination if removed at source
RSYNC_OPTIONS="-avzh --delete"

# Run rsync
rsync $RSYNC_OPTIONS "$SRC_DIR" "$DEST_USER@$DEST_HOST:$DEST_DIR"

# Optional logging
echo "$(date): Rsync from $SRC_DIR to $DEST_USER@$DEST_HOST:$DEST_DIR completed." >> /var/log/rsync_sync.log
```

**How it works:**
- `SRC_DIR` - The local directory to sync
- `DEST_USER` and `DEST_HOST` - Credentials for the remote server
- **Rsync options:**
  - `-a` - Archive mode (preserves permissions, timestamps, symlinks)
  - `-v` - Verbose output
  - `-z` - Compress data during transfer
  - `-h` - Human-readable output
  - `--delete` - Deletes files at destination that no longer exist at source
- Logging tracks sync operations

**Prerequisites:**
- SSH key-based authentication set up between servers
- `rsync` installed on both systems

**Setup SSH keys:**
```bash
# Generate SSH key (if not exists)
ssh-keygen -t rsa -b 4096

# Copy to remote server
ssh-copy-id remoteuser@remote.server.com
```

**Setup cron job:**
```bash
# Run daily at 2 AM
0 2 * * * /path/to/server_sync.sh
```

**Usage:**
```bash
# Edit source and destination paths in the script
./server_sync.sh
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - feel free to use these scripts in your own projects.

---

**Note:** Always test scripts in a safe environment before running in production. Some scripts require root privileges and can modify system files.
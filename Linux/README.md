 1. Write a script to monitor memory/CPU usage and log it every 5 mins.
 I’d use commands like top, vmstat, or free to check CPU/memory usage and append the output to a log file. To run it every 5 minutes, I’d schedule it with cron.”

#!/bin/bash
date >> /var/log/sys_usage.log
top -b -n1 | head -5 >> /var/log/sys_usage.log
free -h >> /var/log/sys_usage.log


date >> /var/log/sys_usage.log
Runs the date command (prints current date and time).

>> appends the output to /var/log/sys_usage.log.

This makes it clear in the log when the system usage was recorded.

bash
Copy code
top -b -n1 | head -5 >> /var/log/sys_usage.log
top shows running processes and CPU/memory usage.

-b → batch mode (so it outputs text instead of interactive UI).

-n1 → run it only once (not continuous).

| head -5 → take only the first 5 lines, which usually include CPU load, tasks, and top resource usage summary.

>> /var/log/sys_usage.log → append this summary to the log file.

bash
Copy code
free -h >> /var/log/sys_usage.log
free shows memory usage (RAM, swap).

-h → human-readable format (MB/GB instead of bytes).

Output is appended to the same log file.

And add to cron:

*/5 * * * * /path/to/script.sh

 2. Create a script to find and delete files older than 30 days.

 #!/bin/bash

TARGET_DIR="/path/to/directory"

find "$TARGET_DIR" -type f -mtime +30 -exec rm -f {} \;

echo "$(date): Deleted files older than 30 days in $TARGET_DIR" >> /var/log/cleanup.log


find "$TARGET_DIR" → searches inside the target directory.

-type f → restricts to files (ignores directories).

-mtime +30 → matches files modified more than 30 days ago.

-exec rm -f {} \; → deletes each file found ({} is placeholder for filename).

>> /var/log/cleanup.log → records the cleanup action for auditing.

 3. Automate parsing an Apache/Nginx log file to find top 10 requested URLs.

 #!/bin/bash

LOGFILE="/var/log/nginx/access.log"

echo "Top 10 requested URLs:"
awk '{print $7}' "$LOGFILE" | sort | uniq -c | sort -nr | head -10


Explanation

awk '{print $7}' "$LOGFILE" → Extracts the 7th field from each line (in common Apache/Nginx log formats, that’s the requested URL path).

sort → Sorts all URLs.

uniq -c → Counts how many times each unique URL appears.

sort -nr → Sorts numerically in reverse (highest first).

head -10 → Displays the top 10 most requested URLs.

 4. Write a script that accepts username and checks if user exists; if not, create
 it.
 5. Automate backup of 
/etc/ directory with date-based filenames.
 6. Create a menu-driven script to manage users (add, delete, list).
 7. Write a script that fetches weather info using 
curl from an API.
 8. Implement a script to validate IP addresses using regex.
 9. Write a script to monitor running services (like ssh, docker) and restart if
 down.
 10. Build a cron job script that syncs files between two servers using 
Terraform
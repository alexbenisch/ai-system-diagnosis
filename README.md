# ai-system-diagnosis
Linux System Diagnostic Report Generator Creates a comprehensive report for offline analysis Uses only POSIX-standard tools for maximum compatibility

```bash
#!/bin/bash
#
# Linux System Diagnostic Report Generator
# Creates a comprehensive report for offline analysis
# Uses only POSIX-standard tools for maximum compatibility
#
# Usage: ./sysdiag.sh [output_file]
# Output: sysdiag_report_YYYYMMDD_HHMMSS.txt
#

set -u

# Output file
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTPUT="${1:-sysdiag_report_${TIMESTAMP}.txt}"

# Helper function to add section headers
section() {
    echo "" >> "$OUTPUT"
    echo "==================================================================================" >> "$OUTPUT"
    echo "= $1" >> "$OUTPUT"
    echo "==================================================================================" >> "$OUTPUT"
    echo "" >> "$OUTPUT"
}

# Helper function to run command with error handling
run_cmd() {
    local cmd="$1"
    local desc="$2"
    echo "--- $desc ---" >> "$OUTPUT"
    eval "$cmd" >> "$OUTPUT" 2>&1 || echo "ERROR: Command failed or not available" >> "$OUTPUT"
    echo "" >> "$OUTPUT"
}

# Start report
{
    echo "LINUX SYSTEM DIAGNOSTIC REPORT"
    echo "Generated: $(date)"
    echo "Hostname: $(hostname)"
    echo "Report file: $OUTPUT"
} > "$OUTPUT"

# 1. SYSTEM INFORMATION
section "1. SYSTEM INFORMATION"
run_cmd "uname -a" "Kernel and OS Info"
run_cmd "cat /etc/os-release 2>/dev/null || cat /etc/redhat-release 2>/dev/null || cat /etc/debian_version 2>/dev/null" "Distribution Info"
run_cmd "uptime" "System Uptime and Load"
run_cmd "date" "Current Date/Time"
run_cmd "who" "Logged in Users"
run_cmd "last -10" "Recent Login History"

# 2. CPU ANALYSIS
section "2. CPU ANALYSIS"
run_cmd "cat /proc/cpuinfo | grep -E 'processor|model name|cpu MHz|cache size' | head -20" "CPU Information"
run_cmd "cat /proc/loadavg" "Load Average (1/5/15 min)"
run_cmd "top -b -n 1 | head -20" "Top Processes Snapshot"
run_cmd "ps aux --sort=-%cpu | head -15" "Top CPU Consumers"
run_cmd "ps aux | wc -l" "Total Process Count"
run_cmd "ps aux | awk '\$8==\"Z\"' | head -10" "Zombie Processes"
run_cmd "ps aux | awk '\$8~\"D\"' | head -10" "Uninterruptible Sleep (D-state) Processes"

# 3. MEMORY ANALYSIS
section "3. MEMORY ANALYSIS"
run_cmd "free -m" "Memory Usage (MB)"
run_cmd "cat /proc/meminfo | grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty'" "Detailed Memory Info"
run_cmd "ps aux --sort=-%mem | head -15" "Top Memory Consumers"
run_cmd "cat /proc/sys/vm/swappiness" "Swappiness Value"
run_cmd "grep -i 'out of memory\\|kill' /var/log/messages 2>/dev/null | tail -20 || dmesg | grep -i 'out of memory\\|kill' | tail -20" "OOM Killer Events"

# 4. DISK ANALYSIS
section "4. DISK ANALYSIS"
run_cmd "df -h" "Disk Space Usage"
run_cmd "df -i" "Inode Usage"
run_cmd "mount" "Mounted Filesystems"
run_cmd "cat /proc/mounts" "Active Mounts"
run_cmd "du -sh /* 2>/dev/null | sort -rh | head -15" "Top Directory Sizes"
run_cmd "find / -type f -size +100M 2>/dev/null | head -20" "Large Files (>100MB)"
run_cmd "lsof +L1 2>/dev/null | head -20 || echo 'lsof not available'" "Deleted but Open Files"

# 5. DISK I/O
section "5. DISK I/O"
run_cmd "cat /proc/diskstats" "Disk Statistics"
run_cmd "grep -E '(r/s|w/s|await)' /proc/diskstats 2>/dev/null || echo 'Limited disk stats available'" "I/O Metrics"

# 6. NETWORK ANALYSIS
section "6. NETWORK ANALYSIS"
run_cmd "ip addr show 2>/dev/null || ifconfig -a" "Network Interfaces"
run_cmd "ip route show 2>/dev/null || route -n" "Routing Table"
run_cmd "cat /etc/resolv.conf" "DNS Configuration"
run_cmd "ss -tuln 2>/dev/null | head -30 || netstat -tuln | head -30" "Listening Ports"
run_cmd "ss -tan 2>/dev/null | awk '{print \$1}' | sort | uniq -c || netstat -tan | awk '{print \$6}' | sort | uniq -c" "Connection States Summary"
run_cmd "ss -s 2>/dev/null || netstat -s | head -50" "Network Statistics Summary"
run_cmd "cat /proc/net/dev" "Network Device Statistics"
run_cmd "ping -c 3 8.8.8.8" "External Connectivity Test (Google DNS)"
run_cmd "ping -c 3 $(ip route | grep default | awk '{print $3}' | head -1) 2>/dev/null" "Gateway Connectivity Test"

# 7. SYSTEM LOGS
section "7. SYSTEM LOGS - KERNEL"
run_cmd "dmesg | tail -100" "Recent Kernel Messages (last 100)"
run_cmd "dmesg | grep -i 'error\\|fail\\|warning\\|critical' | tail -50" "Kernel Errors/Warnings"

section "7. SYSTEM LOGS - SYSLOG"
run_cmd "tail -100 /var/log/syslog 2>/dev/null || tail -100 /var/log/messages 2>/dev/null || echo 'Syslog not accessible'" "Recent Syslog (last 100 lines)"
run_cmd "grep -i 'error\\|fail\\|critical' /var/log/syslog 2>/dev/null | tail -50 || grep -i 'error\\|fail\\|critical' /var/log/messages 2>/dev/null | tail -50 || echo 'Syslog not accessible'" "Recent Syslog Errors"

section "7. SYSTEM LOGS - AUTH"
run_cmd "tail -50 /var/log/auth.log 2>/dev/null || tail -50 /var/log/secure 2>/dev/null || echo 'Auth log not accessible'" "Recent Authentication Events"
run_cmd "grep -i 'failed\\|failure' /var/log/auth.log 2>/dev/null | tail -30 || grep -i 'failed\\|failure' /var/log/secure 2>/dev/null | tail -30 || echo 'Auth log not accessible'" "Failed Login Attempts"

# 8. SERVICES (systemd)
section "8. SERVICES STATUS"
run_cmd "systemctl status 2>/dev/null | head -50 || service --status-all 2>/dev/null | head -50 || echo 'Service status not available'" "Service Overview"
run_cmd "systemctl list-units --failed 2>/dev/null || echo 'systemctl not available'" "Failed Services"
run_cmd "systemctl list-units --state=running 2>/dev/null | head -30 || ps aux | grep -v grep | head -30" "Running Services"

# 9. CRON & SCHEDULED TASKS
section "9. SCHEDULED TASKS"
run_cmd "crontab -l 2>/dev/null || echo 'No user crontab'" "User Crontab"
run_cmd "cat /etc/crontab 2>/dev/null || echo 'No system crontab'" "System Crontab"
run_cmd "ls -la /etc/cron.* 2>/dev/null || echo 'No cron directories found'" "Cron Directories"

# 10. OPEN FILES & LIMITS
section "10. OPEN FILES & RESOURCE LIMITS"
run_cmd "cat /proc/sys/fs/file-nr" "System-wide Open Files (open/free/max)"
run_cmd "ulimit -a" "Current Shell Resource Limits"
run_cmd "lsof 2>/dev/null | wc -l || echo 'lsof not available - cannot count open files'" "Total Open Files Count"
run_cmd "lsof -u root 2>/dev/null | wc -l || echo 'lsof not available'" "Root User Open Files"

# 11. PERFORMANCE INDICATORS
section "11. PERFORMANCE INDICATORS"
run_cmd "cat /proc/stat | grep '^cpu '" "Overall CPU Stats"
run_cmd "cat /proc/vmstat | grep -E 'pgpgin|pgpgout|pswpin|pswpout'" "Paging Activity"
run_cmd "cat /proc/pressure/cpu 2>/dev/null || echo 'PSI (Pressure Stall Information) not available'" "CPU Pressure"
run_cmd "cat /proc/pressure/memory 2>/dev/null || echo 'Memory Pressure not available'" "Memory Pressure"
run_cmd "cat /proc/pressure/io 2>/dev/null || echo 'I/O Pressure not available'" "I/O Pressure"

# 12. SECURITY INDICATORS
section "12. SECURITY INDICATORS"
run_cmd "last -20" "Recent Logins (last 20)"
run_cmd "lastb -20 2>/dev/null || echo 'Bad login history not accessible'" "Failed Logins (last 20)"
run_cmd "ps aux | grep -E 'ssh|sshd' | grep -v grep" "SSH Processes"
run_cmd "find / -perm -4000 -type f 2>/dev/null | head -20" "SUID Files (first 20)"

# 13. ENVIRONMENT
section "13. ENVIRONMENT VARIABLES"
run_cmd "env | sort" "Environment Variables"
run_cmd "echo \$PATH" "PATH Variable"

# 14. HARDWARE INFO
section "14. HARDWARE INFORMATION"
run_cmd "cat /proc/cpuinfo | grep -E 'processor|vendor_id|model name' | head -20" "CPU Details"
run_cmd "cat /proc/meminfo | grep MemTotal" "Total Physical Memory"
run_cmd "ls -l /sys/class/net/" "Network Interfaces List"
run_cmd "cat /proc/devices" "Configured Devices"

# 15. QUICK DIAGNOSTICS SUMMARY
section "15. QUICK DIAGNOSTICS SUMMARY"
{
    echo "### POTENTIAL ISSUES DETECTED ###"
    echo ""
    
    # Check high load
    LOAD=$(cat /proc/loadavg | awk '{print $1}')
    CPU_COUNT=$(grep -c ^processor /proc/cpuinfo)
    echo "CPU Count: $CPU_COUNT"
    echo "Current Load (1min): $LOAD"
    
    # Check memory
    MEM_FREE=$(free | grep Mem | awk '{print $4}')
    MEM_TOTAL=$(free | grep Mem | awk '{print $2}')
    echo "Free Memory: $MEM_FREE KB of $MEM_TOTAL KB"
    
    # Check disk
    echo ""
    echo "Filesystems over 80% full:"
    df -h | awk '$5+0 > 80 {print $0}'
    
    # Check zombies
    echo ""
    ZOMBIES=$(ps aux | awk '$8=="Z"' | wc -l)
    echo "Zombie Processes: $ZOMBIES"
    
    # Check D-state
    DSTATE=$(ps aux | awk '$8~"D"' | wc -l)
    echo "D-state Processes: $DSTATE"
    
    # Check failed services
    echo ""
    echo "Failed Services:"
    systemctl list-units --failed 2>/dev/null || echo "Cannot check (systemctl not available)"
    
} >> "$OUTPUT"

# Completion
{
    echo ""
    echo "==================================================================================="
    echo "Report generation completed: $(date)"
    echo "Output file: $OUTPUT"
    echo "File size: $(du -h "$OUTPUT" | cut -f1)"
    echo "==================================================================================="
} >> "$OUTPUT"

echo "System diagnostic report generated: $OUTPUT"
echo "File size: $(du -h "$OUTPUT" | cut -f1)"
echo ""
echo "To analyze with Claude-code:"
echo "  1. Download this file"
echo "  2. Run: claude-code"
echo "  3. Upload the report and ask: 'Analyze this system diagnostic report and identify issues'"
```
Claude-code Analyse-Prompts:
Nachdem du den Report generiert hast, nutze diese Prompts in Claude-code:
# Basis-Analyse
Analyze this system diagnostic report and identify the top 5 issues with their severity and recommended actions.

# Detaillierte Performance-Analyse
Based on this diagnostic report, analyze the performance bottlenecks. Focus on CPU, memory, disk I/O, and network. Provide specific metrics and recommendations.

# Sicherheits-Audit
Review this report for security concerns including failed logins, open ports, SUID files, and unusual processes.

# Kapazitätsplanung
Based on the current resource usage in this report, project when we'll need to scale and what resources to add.

# Root-Cause-Analyse
The system is experiencing [describe symptom]. Analyze this diagnostic report to find the root cause.
Quick Test:
chmod +x sysdiag.sh
./sysdiag.sh
cat sysdiag_report_*.txt | less



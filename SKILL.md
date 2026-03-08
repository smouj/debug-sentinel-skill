---
name: Debug Sentinel
slug: debug-sentinel
description: Real-time security monitoring and anomaly detection for OpenClaw applications with runtime threat analysis
version: 1.2.0
author: Security Team (OpenClaw)
tags: monitoring, threat-detection, runtime-analysis, security
requires: [python3 >=3.8, psutil, scapy, elasticsearch==7.17.0, kibana, docker, libpcap-dev]
conflicts: [legacy-monitor, old-audit]
priority: 90
env_vars:
  - DEBUG_SENTINEL_CONFIG: Path to YAML config (default: ~/.config/debug-sentinel/config.yaml)
  - SENTINEL_LOG_LEVEL: DEBUG, INFO, WARNING, ERROR (default: INFO)
  - SENTINEL_DB_PATH: Elasticsearch connection string (default: http://localhost:9200)
  - SENTINEL_ALERT_WEBHOOK: Slack/Discord webhook for critical alerts
---

# Debug Sentinel

Runtime security monitoring for OpenClaw applications with real-time threat detection, anomaly analysis, and forensic logging.

## Purpose

**Real use cases:**
- Detect SQL injection attacks in OpenClaw API endpoints by monitoring query patterns and payload signatures
- Identify privilege escalation attempts via suspicious process trees and capabilities
- Monitor abnormal network connections from containerized applications (Docker/K8s)
- Alert on kernel-level anomalies like unexpected syscalls from application processes
- Automated forensic timeline generation for incident response
- Compliance logging for SOC2/PCI requirements with immutable audit trails
- Performance anomaly detection correlating security events with resource spikes

## Scope

**Exact commands available:**

### Core Commands
```
debug-sentinel start [--config PATH] [--daemon] [--pid-file PATH]
debug-sentinel stop [--graceful] [--timeout SECONDS]
debug-sentinel status [--json] [--verbose] [--metrics]
debug-sentinel restart [--config PATH]
```

### Analysis Commands
```
debug-sentinel analyze [--since TIMESPEC] [--until TIMESPEC] [--app APP_NAME] [--severity LEVEL]
                        [--format {json,table,csv}] [--output FILE]
debug-sentinel alert list [--active] [--resolved] [--severity {CRITICAL,HIGH,MEDIUM,LOW}]
debug-sentinel alert ack ALERT_ID [--comment TEXT]
debug-sentinel alert resolve ALERT_ID [--auto]
debug-sentinel threat-hunt [--ioc IOC_VALUE] [--type {ip,domain,hash,user}] [--hours N]
```

### Configuration Commands
```
debug-sentinel rules list [--enabled] [--category CATEGORY]
debug-sentinel rules add RULE_FILE [--category CATEGORY] [--priority 1-10]
debug-sentinel rules remove RULE_ID [--force]
debug-sentinel rules test RULE_FILE [--sample-traffic PATH]
debug-sentinel config show [--section SECTION]
debug-sentinel config set SECTION.KEY VALUE [--type {string,int,bool,list}]
debug-sentinel config validate
```

### Dashboard & Reporting
```
debug-sentinel dashboard [--host HOST] [--port PORT] [--ssl] [--auth USER:PASS]
debug-sentinel report generate [--type {daily,weekly,incident}] [--since DATE] [--output PATH]
debug-sentinel export evidence ALERT_ID [--format {json,zip}] [--output PATH]
```

### Maintenance
```
debug-sentinel db cleanup [--older-than DAYS] [--dry-run]
debug-sentinel index rotate [--daily|--weekly] [--keep N]
debug-sentinel healthcheck [--verbose]
debug-sentinel self-test [--all] [--category {network,syscall,file,process}]
```

## Detailed Work Process

**Real operational workflow:**

### 1. Initial Setup
```bash
# Create config directory
mkdir -p ~/.config/debug-sentinel/

# Generate default production config
debug-sentinel config generate --template production > ~/.config/debug-sentinel/config.yaml

# Edit critical sections:
# - elasticsearch.hosts: ["http://es-cluster:9200"]
# - monitoring.apps: ["rpg-claw-api", "rpg-claw-worker"]
# - rules.directories: ["/opt/debug-sentinel/rules/enabled", "/opt/debug-sentinel/rules/custom"]
# - alerting.webhooks: ["https://hooks.slack.com/services/..."]
# - network.interfaces: ["eth0", "docker0"]

# Validate configuration syntax
debug-sentinel config validate
```

### 2. Rule Deployment
```bash
# Download latest detection rules from OpenClaw security repo
git clone https://github.com/openclaw/security-rules /opt/debug-sentinel/rules

# Enable only production-tested rules
cd /opt/debug-sentinel/rules/enabled
ln -sf ../available/sql_injection.yaml .
ln -sf ../available/path_traversal.yaml .
ln -sf ../available/privilege_escalation.yaml .

# Test rule syntax without deploying
debug-sentinel rules test /opt/debug-sentinel/rules/custom/my_rule.yaml

# Add custom rule for known attack pattern
debug-sentinel rules add /opt/debug-claw/rules/custom/rpg_claw_specific.yaml \
  --category "application" \
  --priority 8

# Verify rule count
debug-sentinel rules list | wc -l  # Should show 150+ rules
```

### 3. Daemon Deployment
```bash
# Start as systemd service (recommended)
sudo cp debug-sentinel.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable debug-sentinel
sudo systemctl start debug-sentinel

# Verify service status with metrics
debug-sentinel status --verbose --metrics

# Expected output:
# State: RUNNING (PID 2847)
# Uptime: 2h 15m
# Events processed: 1,247,893
# Rules loaded: 167
# Alerts (24h): CRITICAL:3, HIGH:12, MEDIUM:47, LOW:156
# Memory: 312MB, CPU avg: 2.4%
# Last heartbeat: 2025-03-08T10:45:22Z
```

### 4. Continuous Monitoring
```bash
# Real-time tail of security events (like tail -f)
debug-sentinel analyze --since 5m --format json | jq '.[] | select(.severity=="CRITICAL")'

# Check active alerts requiring attention
debug-sentinel alert list --active --severity HIGH

# Sample output:
# ID: ALT-7f3a9b2c  Severity: HIGH  Rule: SQL_INJECTION_DETECTED
# App: rpg-claw-api  Source: 192.168.1.105  Time: 10:42:15
# Evidence: "SELECT * FROM users WHERE id='1' OR '1'='1'"
# [ACK] [RESOLVE]

# Threat hunt for specific IoC across 30 days
debug-sentinel threat-hunt --ioc "35.200.220.123" --type ip --hours 720

# Generate forensic timeline for incident ALT-7f3a9b2c
debug-sentinel export evidence ALT-7f3a9b2c --format json --output /tmp/incident_7f3a.json
```

### 5. Incident Response
```bash
# When CRITICAL alert fires:
# 1. Immediate check
debug-sentinel alert list --active --severity CRITICAL

# 2. Export all evidence with context
debug-sentinel export evidence ALT-xxxx --format zip --output /tmp/forensic_$(date +%s).zip

# 3. Analyze process tree at time of incident
debug-sentinel analyze --since "2025-03-08T10:40" --until "2025-03-08T10:45" \
  --format json | jq '.[] | select(.event_type=="process_exec")'

# 4. Generate incident report
debug-sentinel report generate --type incident --since "2025-03-08" --output /tmp/report.pdf

# 5. Acknowledge and track
debug-sentinel alert ack ALT-xxxx --comment "IR team investigating, isolating affected host"
```

## Golden Rules

1. **Rule Testing Mandatory**: All custom rules MUST pass `debug-sentinel rules test` with 0 false positives on 1000+ sample legitimate requests before deployment to production

2. **Threshold Calibration**: Never use default detection thresholds in production. Run in `LEARN` mode for 7 days to baseline legitimate traffic, then switch to `ENFORCE` with tuned thresholds

3. **Alert Fatigue Prevention**: Configure severity filters so HIGH+ alerts trigger pagerduty, MEDIUM goes to Slack #security, LOW to daily digest. Weekly review of false positives required

4. **Immutable Logging**: Elasticsearch indices must have `index.lifecycle.rollover_alias: debug-sentinel-logs` and `number_of_shards: 1`. Never delete raw logs manually; use `debug-sentinel index rotate`

5. **Network Capture Scope**: Only monitor interfaces explicitly listed in config. Never enable promiscuous mode on production without network team approval

6. **Response Automation Limit**: Auto-blocking (fail2ban integration) ONLY for verified attack patterns with <0.1% false positive rate over 30 days

7. **Privacy Compliance**: Mask PII in logs using redaction rules before storage. Enabled by default: `redaction.enabled: true` with standard patterns for emails, SSNs, credit cards

8. **Performance Budget**: Sentinel daemon must not exceed 5% CPU or 500MB RAM on production hosts. Sample accordingly: `sampling.rate: 0.1` for high-traffic (>10k RPS) apps

## Examples

### Example 1: Detect SQL Injection in RPGClaw API
```bash
# User input: Attacker sends malicious payload
curl -X POST https://rpg-claw.example.com/api/user/search \
  -H "Authorization: Bearer <valid-token>" \
  -d '{"query": "1' OR '1'='1"}'

# Debug Sentinel detection (15ms later):
$ debug-sentinel alert list --active --severity HIGH
ID: ALT-a1b2c3d4  Severity: HIGH  Rule: SQLI_PATTERN_COMMON
Time: 2025-03-08 10:42:15 UTC
App: rpg-claw-api
Source IP: 192.168.1.105 (attacker.example.com)
User: authenticated_user_45
Evidence:
  Request ID: req_7f8g9h0j
  Payload: {"query": "1' OR '1'='1'"}
  Matched pattern: OR '1'='1'
  Stack trace: /app/api/search.py:127
  Process tree: nginx(2847)#python3(2851)#handler(2852)

# Automated response if configured:
# - IP blocked via fail2ban (sentinel.jail.conf)
# - User session revoked (API callback)
# - Slack alert to #secops
```

### Example 2: Process Privilege Escalation Detection
```bash
# Event: Application process tries to elevate privileges
$ debug-sentinel analyze --since 1h --format json | jq '.[] | select(.event=="setuid")'
{
  "timestamp": "2025-03-08T09:15:33Z",
  "event_type": "setuid",
  "severity": "CRITICAL",
  "app": "rpg-claw-worker",
  "pid": 3456,
  "uid_before": 1001,
  "uid_after": 0,
  "process": "/usr/bin/python3 /opt/rpgclaw/worker/tasks.py",
  "parent_process": "celery(2890)",
  "rule_id": "PRIV_ESC_SETUID_0",
  "container_id": "a1b2c3d4e5f6"
}

# Response: Immediately kill process and isolate container
docker stop rpg-claw-worker-01
nsenter -t 3456 -m -u -i -n -p kill -9 3456
```

### Example 3: Network Anomaly - Data Exfiltration
```bash
# Detect abnormal outbound connections from app container
$ debug-sentinel alert list --severity HIGH --since 6h
ID: ALT-z9y8x7w6  Severity: HIGH  Rule: UNUSUAL_OUTBOUND
App: rpg-claw-api
Time: 2025-03-08 08:30:00 UTC
Evidence:
  External IP: 185.220.101.45 (Russia - unapproved region)
  Port: 4444/TCP
  Bytes sent: 2.4MB in 5 minutes
  Process: python3(zip_export.py)
  Historical baseline: <50KB/day average for this process
  GDPR impact: HIGH (customer data location violation)

# Check geo IP database loaded
debug-sentinel config show geoip
# Output: geoip.enabled: true, database: /var/lib/geoip/GeoLite2-City.mmdb
```

### Example 4: Runtime Threat Hunt
```bash
# After detecting suspicious user "admin_temp" created at 04:00
$ debug-sentinel threat-hunt --ioc "admin_temp" --type user --hours 24

HUNT RESULTS for IoC: admin_temp (user)
========================================
Found 47 events across 3 hosts:

1. 2025-03-08 04:02:11 - Host: rpg-db-01
   Event: USER_ADDED (auditd)
   Details: uid=1002, gid=1002, home=/home/admin_temp
   Session: ssh from 10.0.5.23 (jumpbox)

2. 2025-03-08 04:03:45 - Host: rpg-claw-api-02
   Event: SUDO_EXEC
   Details: admin_temp -> root via ALL NOPASSWD
   Command: systemctl restart nginx

3. 2025-03-08 04:05:22 - Host: rpg-claw-api-02
   Event: FILE_ACCESS /etc/shadow
   Severity: CRITICAL

RECOMMENDATION: Immediate isolation of rpg-claw-api-02, credential rotation for all admin_temp SSH keys
```

### Example 5: Kubernetes Environment Monitoring
```bash
# Deploy as DaemonSet in K8s cluster
kubectl apply -f debug-sentinel-daemonset.yaml

# Check pod status
kubectl get pods -n security -l app=debug-sentinel

# Query alerts from specific namespace
debug-sentinel alert list --filter 'kubernetes.namespace=="rpg-claw-prod"' \
  --since 1h --format table

# Extract container security events
debug-sentinel analyze --app "rpg-claw-frontend" \
  --since 30m --format json | jq -r '.container_id' | sort -u
```

## Rollback Commands

**Immediate emergency rollback:**
```bash
# Stop monitoring completely (use only for active compromise)
debug-sentinel stop --graceful --timeout 10  # Graceful drain first
# If hangs:
pkill -f debug-sentinel
# Verify stopped:
debug-sentinel status  # Should show "STOPPED"

# Remove all iptables rules injected by sentinel
debug-sentinel iptables flush  # Only if integrated with fail2ban
# Or manually:
iptables -D INPUT -p tcp --dport 22 -j DROP  # Remove blocked IPs
```

**Rollback configuration:**
```bash
# Restore previous config from auto-backup (created on every config change)
cp ~/.config/debug-sentinel/config.backup.{timestamp}.yaml \
   ~/.config/debug-sentinel/config.yaml

# Or revert to known-good version from git
git -C /opt/debug-sentinel/rules checkout HEAD -- enabled/
debug-sentinel rules reload

# Restart with previous baseline
debug-sentinel restart --config /opt/debug-sentinel/configs/prod-baseline.yaml
```

**Database cleanup rollback:**
```bash
# If you ran debug-sentinel db cleanup by mistake
# Check what would be deleted first next time:
debug-sentinel db cleanup --older-than 90 --dry-run  # Shows count
# If you deleted indices and need restore:
# 1. Find snapshot from before deletion
curl -X GET "localhost:9200/_snapshot/debug-sentinel-backup/snap*/_all" | jq .
# 2. Restore specific index
curl -X POST "localhost:9200/_snapshot/debug-sentinel-backup/snap_20250305/_restore" \
  -H 'Content-Type: application/json' -d '{"indices": "logs-2025.03.01"}'
```

**Disable specific detection temporarily:**
```bash
# Suspend rule without removing (for false positive investigation)
debug-sentinel rules disable RULE_ID --comment "False positive on payment processing"
# Re-enable after fix:
debug-sentinel rules enable RULE_ID
```

**Full uninstall with artifact cleanup:**
```bash
# Stop service
sudo systemctl stop debug-sentinel
sudo systemctl disable debug-sentinel

# Remove configs (keep backups first!)
cp -r ~/.config/debug-sentinel ~/.config/debug-sentinel.bak
rm -rf ~/.config/debug-sentinel

# Remove rules and database (CAUTION: deletes all historical data)
# Keep if you need compliance archives!
# rm -rf /opt/debug-sentinel/
# curl -X DELETE "localhost:9200/debug-sentinel-*"  # Elasticsearch wipe
```

**Rollback checklist after incident:**
1. `debug-sentinel stop` - isolate monitoring to prevent further alerts
2. Review `/var/log/debug-sentinel/audit.log` for sentinel's own actions
3. Restore config from timestamped backup: `config.backup.YYYYMMDD_HHMM.yaml`
4. `debug-sentinel start --config restored_config.yaml`
5. Validate: `debug-sentinel healthcheck --verbose`
6. Manually verify critical detections still functional: `debug-sentinel self-test --category network`
7. Document root cause in ~/.config/debug-sentinel/incidents/INC-YYYY-NNN.md

**Partial rollback scenarios:**

*False positive flood:*
```bash
# Disable specific rule causing issues
debug-sentinel rules disable SQLI_PATTERN_COMMON --comment "False positive on legacy client"
# Or adjust threshold
debug-sentinel config set detection.rules.sql_injection.threshold 0.95
debug-sentinel restart
```

*Performance degradation:*
```bash
# Reduce sampling rate
debug-sentinel config set sampling.rate 0.05  # From 0.1
# Disable non-critical rule categories
debug-sentinel config set rules.disabled_categories "["performance","compliance"]"
debug-sentinel restart
```

*Alert channel failure:*
```bash
# Disable webhook temporarily
debug-sentinel config set alerting.webhooks "[]"
# Or route to backup channel
debug-sentinel config set alerting.webhooks '["https://backup-hooks..."]'
debug-sentinel restart
```
```
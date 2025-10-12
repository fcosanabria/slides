---
title: Systemd **Fundamentals Workshop**
sub_title: 2 Sessions - Linux System Management
author: Francisco Sanabria
date: 2025

theme:
  override:
    footer:
      style: template
      center: "Systemd Fundamentals Workshop"
      left: "Francisco Sanabria, 2025. CC0"
---

# Systemd Fundamentals Workshop

**2 Sessions - 4 Hours Total**

- Session 1: Fundamentals and Basic Management
- Session 2: Advanced Configuration and Automation

<!-- end_slide -->

# Francisco Sanabria

Site Reliability Engineer @ Datasite

- 10+ years of experience in IT
- IBM: Cloud Engineer
- Datasite: Site Reliability Engineer
- Opensource ❤️
- Professional Nerd
  - Homelab
  - Visual Artist

### Contact

- linkedin.com/in/fcosanabria
- github.com/fcosanabria
- instagram.com/digital.death.disrupt

<!-- end_slide -->


# **SESSION 2**

## Advanced Configuration & Automation

- Deep dive into advanced unit files
- Additional unit types (timer, socket, mount, path)
- Security hardening & environment management
- Templates & multi-instance services
- Timers, scheduling & socket activation
- Monitoring, analysis & lightweight alerting
- Hands-on advanced exercises

<!-- end_slide -->

# Recap From Session 1

You should now be comfortable with:
- Managing services (start/stop/enable)
- Reading logs with `journalctl`
- Creating a simple `.service`
- Basic troubleshooting states

Today we level up → production-grade patterns.

<!-- end_slide -->

# Advanced Unit File Anatomy

Sections revisited:
- `[Unit]` -  metadata + ordering + conditions
- `[Service]` - runtime behavior & exec control
- `[Install]` - integration with targets

Key new directives:
- Dependencies: `Requires=` `Wants=` `BindsTo=`
- Ordering: `After=` `Before=`
- Conditions: `ConditionPathExists=` `ConditionKernelCommandLine=`
- Assertions: `AssertPathIsDirectory=` etc.

<!-- speaker_note: | 

Emphasize difference between dependency (Requires/Wants) and ordering (After/Before). Conditions gate activation pre-start; they do not create dependencies. Assertions fail the unit if false; conditions skip. 

Unit Sections

• [Unit]: Who/when – name, description, what must be ready first, optional conditions to decide “should I even try?”
• [Service]: How to run – command, user, restart rules, security limits, environment, working directory.
• [Install]: How to plug it into the system – which target (boot phase/group) wants it when you enable it.

-->

<!-- end_slide -->

# Dependency vs Ordering

| Purpose | Dependency | Ordering |
|---------|------------|----------|
| Guarantee presence | `Requires=` | (none) |
| Soft relationship | `Wants=` | (none) |
| Lifecycle link | `BindsTo=` | (none) |
| Start order | (none) | `After=` / `Before=` |
| Stop behavior | `Requires=` stops dependents | Ordering does not |

Example:
```
[Unit]
Requires=network-online.target
After=network-online.target
Wants=syslog.service
```

<!-- speaker_note: | 

Dependencies vs Ordering

• Requires=: “I can’t run unless that other thing exists; if it stops I stop.”
• Wants=: “Nice to have; try to start it, but I still run if it fails.”
• After=/Before=: “Timing only” (who starts first) — no guarantee the other succeeds.
• BindsTo=: Strong leash: if the bound unit dies, this one is stopped too.

-->

<!-- end_slide -->

# Conditions & Assertions

Skip vs Fail:
- `Condition*=` → if false: unit skipped (EXIT 0)
- `Assert*=` → if false: unit fails

Examples:
```
ConditionPathExists=/etc/myapp/config.yaml
ConditionKernelCommandLine=debug
AssertPathIsDirectory=/var/lib/myapp
```
Check evaluation:
```
systemctl show -p Conditions myapp.service
```

<!-- speaker_note: | 

You can also use statements such as ConditionFileNotEmpty, or triggers, using a pipe.


- AssertPathIsDirectory directive in systemd is used within unit files to ensure that a specific path exists and is a directory before the unit is activated.


Conditions vs Assertions

• Condition*= : If false → skip quietly (treated like ‘not needed today’).
• Assert*= : If false → unit fails (red flag).
• Use Conditions for optional prerequisites; Assertions for must-have invariants.


-->


<!-- end_slide -->

# Service Restart Strategies

Key directives:
```
Restart=always|on-failure|on-abnormal|on-watchdog|no
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
StartLimitAction=reboot|reboot-force|none
```

<!-- speaker_note: Restart & Rate Limiting

• Restart=… : When to auto-try again (on-failure, always, etc.).
• RestartSec= : Wait time before reattempt.
• StartLimitIntervalSec= + StartLimitBurst= : “How many crashes allowed in this window?”
• Protects against crash loops.  


    Restart policies are about "when" – when to try again after failure
    Rate limits are about "how many" – preventing infinite loops
    RestartSec is the pause button – gives systems time to recover
    StartLimitBurst + Interval = circuit breaker – 5 tries in 5 minutes is typical
    Watchdog catches hangs, not crashes – requires app cooperation
    StartLimitAction=reboot is a last resort – use for critical-only services
-->

Pattern:
```
[Service]
Restart=on-failure
RestartSec=3
StartLimitIntervalSec=120
StartLimitBurst=10
```

Monitor:
```
journalctl -u myapp --grep=Start.*failed
```

<!-- end_slide -->

# Lifecycle Hooks

Lifecycle hooks:
- `ExecStartPre=` (can have multiple)
- `ExecStart=` (main process)
- `ExecStartPost=`
- `ExecReload=`
- `ExecStop=` / `ExecStopPost=`


<!-- speaker_note: | 

Here, we are going to see the ExecStart variants and the Lifecycle hooks.  

 ExecStartPre=: Checks / setup steps (can be multiple).
 ExecStart=: Main process (exactly one line or a list with ExecStart= per alternative in some cases).
 ExecStartPost=: Runs right after successful start (logging, warm-up).
 ExecReload=: Command to apply new config without full restart (signals).
 ExecStop=/ExecStopPost=: Graceful shutdown + cleanup.

    ExecStartPre = "Can we start?" (validation)
    ExecStart = "The main show" (your app)
    ExecStartPost = "We're live!" (announcements)
    ExecReload = "Update without downtime" (optional)
    ExecStop = "Shut down nicely" (graceful)
    ExecStopPost = "Clean up always" (guaranteed)
    Use "-" prefix for optional steps
    Set timeouts to prevent hangs
    Order matters in Pre/Post hooks
    ExecStopPost runs even on crash – use it for critical cleanup


-->

Example:
```
[Service]
Type=simple
ExecStartPre=/usr/bin/test -f /etc/myapp/config.yml
ExecStart=/usr/local/bin/myapp --config /etc/myapp/config.yml
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
```
<!-- speaker_note | 

    Type=simple → "My app runs in foreground"
    ExecStartPre → "Check prerequisites before starting"
    ExecStart → "This is my app"
    ExecReload → "Send SIGHUP to reload config without restart"
    ExecStop → "Send SIGTERM for graceful shutdown"
    $MAINPID → "systemd's way of saying 'the main process PID'"
    Signals are requests → App must be coded to handle them
    Order matters → Pre → Start → (running) → Reload/Stop

One-Sentence Summary: "Check config exists, start the app, support hot-reload via SIGHUP, and shut down gracefully with SIGTERM."

-->

<!-- end_slide -->

# Important Unit Types (Beyond Service)

| Type | Purpose |
|------|---------|
| `.socket` | On-demand activation via sockets |
| `.timer`  | Scheduled activation (cron substitute) |
| `.path`   | Path change activation |
| `.mount`  | Filesystem mount management |
| `.target` | Synchronization point / grouping |


## Example: 

- Activation pattern: `foo.socket` → triggers `foo.service`.
- Timer pattern: `backup.timer` → triggers `backup.service`.

<!-- end_slide -->

# Path Units


<!-- speaker_note: Think "When file X appears/changes, run service Y." -->

Watch filesystem events:
```
[Unit]
Description=Trigger index rebuild when data updates

[Path]
PathChanged=/var/lib/myapp/data/index.src
Unit=reindex.service

[Install]
WantedBy=multi-user.target
```
Enable & monitor:
```
systemctl enable --now reindex.path
systemctl status reindex.path
```

<!-- speaker_note: |

Common Patters:

1. Auto-reload on Config Change
2. Process Upload Queue
3. Trigger on File Creation
4. Watch Multiple Configs

Real-World Example: Certificate Renewal

Problem: Let's Encrypt renews certs, need to reload nginx.
Result: Cert renews → nginx reloads automatically → zero manual intervention.
-->


<!-- end_slide -->

# Mount Units

Auto-managed by systemd; name derived from path (`/data/app` → `data-app.mount`).
```
[Unit]
Description=App Data Mount

[Mount]
What=/dev/mapper/vg0-app
Where=/data/app
Type=ext4
Options=noatime

[Install]
WantedBy=multi-user.target
```
Reload & start:
```
systemctl daemon-reload
systemctl start data-app.mount
```

<!-- end_slide -->

# Environment & Configuration

Options:
- `Environment=KEY=VALUE KEY2=VAL2`
- `EnvironmentFile=/etc/sysconfig/myapp` (one per line)
- `WorkingDirectory=/var/lib/myapp`
- `PassEnvironment=VAR1 VAR2`
- Drop-in override files (`systemctl edit myapp`)

Example override:
```
# /etc/systemd/system/myapp.service.d/env.conf
[Service]
Environment=LOG_LEVEL=debug FEATURE_X=1
```

<!-- end_slide -->

# Security Hardening Basics

Useful directives:
- `User=` / `Group=`
- `CapabilityBoundingSet=`
- `AmbientCapabilities=`
- `NoNewPrivileges=yes`
- `PrivateTmp=yes`
- `ProtectSystem=strict` / `full`
- `ProtectHome=yes`
- `ReadWritePaths=/var/lib/myapp`
- `RestrictAddressFamilies=AF_INET AF_UNIX`
- `SystemCallFilter=@system-service`
- `ProtectKernelLogs=yes`

Principle: deny by default, allow minimal filesystem + capabilities.

<!-- end_slide -->

# Hardening Example

```
[Service]
User=myapp
Group=myapp
ExecStart=/usr/local/bin/myapp
WorkingDirectory=/var/lib/myapp
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/myapp /var/log/myapp
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
RestrictAddressFamilies=AF_INET AF_UNIX
SystemCallFilter=@system-service
```
Check sandbox:
```
systemd-analyze security myapp.service
```

<!-- end_slide -->

# Templates & Multi-Instance Services

Template filename: `app@.service`
Instance name: `app@instance1.service`

Template example:
```
[Unit]
Description=My App Instance %i

[Service]
ExecStart=/usr/local/bin/myapp --instance %i --config /etc/myapp/%i.yml
User=myapp
Group=myapp

[Install]
WantedBy=multi-user.target
```

Start two instances:
```
systemctl start app@blue app@green
```

<!-- end_slide -->

# Template Specifiers (Common)

| Token | Expands To |
|-------|------------|
| `%i`  | Instance name (full) |
| `%I`  | Unescaped instance |
| `%n`  | Unit name |
| `%p`  | Prefix (name before `@`) |
| `%H`  | Hostname |
| `%f`  | Unescaped filename |

Inspect expansion using:
```
systemctl status app@blue
```

<!-- end_slide -->

# Timers vs Cron

Why timers?
- Integrated with unit dependencies
- Persistent (catch-up) execution via `Persistent=true`
- Unified logging & status
- Calendar syntax & monotonic timers
- Randomized delay (`RandomizedDelaySec=`)

Compare:
| Feature | cron | systemd timer |
|---------|------|---------------|
| Logs | manual redirect | journalctl |
| Dependencies | none | full unit model |
| Missed runs | skipped | can catch up |
| Security | runs as root unless specified | per-unit sandbox |

<!-- end_slide -->

# Timer Unit Anatomy

Service: `backup.service`
```
[Unit]
Description=Nightly Backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```
Timer: `backup.timer`
```
[Unit]
Description=Nightly Backup Schedule

[Timer]
OnCalendar=03:15
Persistent=true
RandomizedDelaySec=300
Unit=backup.service

[Install]
WantedBy=timers.target
```
Enable:
```
systemctl enable --now backup.timer
systemctl list-timers --all
```

<!-- end_slide -->

# Calendar Expressions

Examples:
```
OnCalendar=*-*-01 05:00:00      # First day monthly
OnCalendar=Mon..Fri 09:00       # Weekdays at 9 AM
OnCalendar=Mon *-*-* 02:00:00   # Mondays at 2 AM
OnCalendar=Hourly               # Shortcut
OnCalendar=*:0/15               # Every 15 min
OnCalendar=2025-10-01 00:00     # One-off
```
Test parsing:
```
systemd-analyze calendar "Mon..Fri 09:00"
```

<!-- end_slide -->

# Monotonic Timers

Trigger after boot / activation:
```
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
OnUnitInactiveSec=30m
AccuracySec=1min
```
Use with `oneshot` services for periodic jobs without calendar complexity.

<!-- end_slide -->

# Socket Activation

Pattern: defer expensive daemon startup until needed.

Socket unit:
```
[Unit]
Description=MyApp API Socket

[Socket]
ListenStream=9090
Accept=no

[Install]
WantedBy=sockets.target
```
Service unit `myapp.service`:
```
[Service]
ExecStart=/usr/local/bin/myapp --listen fd://0
```
Enable:
```
systemctl enable --now myapp.socket
ss -ltnp | grep 9090
# Connection triggers service start
```

<!-- end_slide -->

# Per-Connection Services (Accept=yes)

Socket:
```
[Socket]
ListenStream=2222
Accept=yes
Service=echod@.service
```
Template service:
```
[Service]
ExecStart=/usr/local/bin/echod %i
StandardInput=socket
```
Each connection spawns `echod@<id>.service`.

<!-- end_slide -->

# On-Demand Path vs Socket vs Timer

| Trigger | Use When |
|---------|----------|
| Socket  | On first network/IPC use |
| Path    | File changed/created |
| Timer   | Time / interval |

Combine patterns for efficient resource usage (e.g., periodic cleanup + on-demand start).

<!-- end_slide -->

# Monitoring & Analysis Tools

Commands:
```
systemd-analyze blame
systemd-analyze critical-chain
systemctl list-dependencies nginx
systemctl show myapp -p ActiveState -p SubState
journalctl -u myapp -p warning --since -1h
```
Security:
```
systemd-analyze security myapp.service
```
Transient units:
```
systemd-run --unit=diag --property=PrivateTmp=yes /usr/bin/id
```

<!-- end_slide -->

# Basic Alerting Pattern

Use `OnFailure=` + notifier service.
```
[Unit]
Description=Core App
OnFailure=notify-admin@%n.service
```
Notifier template:
```
[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify.sh "%i failed" $(hostname)
```
`notify.sh` can send email, Slack webhook, etc.

<!-- end_slide -->

# Watchdog Integration

Add to service:
```
[Service]
WatchdogSec=30s
Type=notify
ExecStart=/usr/local/bin/healthd
```
App periodically calls:
```
systemd-notify --watchdog
```
Systemd restarts if heartbeat stops.

<!-- end_slide -->

# Hands-On 2.1: Restart Policies

Goal: Harden & stabilize `unstable.service`.

1. Create failing script:
```
sudo tee /opt/unstable.sh <<'EOF'
#!/bin/bash
echo "Run at: $(date)"; exit 1
EOF
chmod +x /opt/unstable.sh
```
2. Unit file:
```
[Unit]
Description=Unstable Demo

[Service]
Type=simple
ExecStart=/opt/unstable.sh
Restart=on-failure
RestartSec=2
StartLimitIntervalSec=20
StartLimitBurst=5
```
3. Observe behavior & limits.

<!-- end_slide -->

# Hands-On 2.2: Backup Timer

1. Service:
```
[Unit]
Description=Backup /var/www

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup-www.sh
```
2. Script:
```
sudo tee /usr/local/bin/backup-www.sh <<'EOF'
#!/bin/bash
set -euo pipefail
tar -czf /backup/www-$(date +%F-%H%M).tgz /var/www
EOF
chmod +x /usr/local/bin/backup-www.sh
```
3. Timer:
```
[Timer]
OnCalendar=*-*-* 02:30
Persistent=true
RandomizedDelaySec=600
Unit=backup.service
```
4. Enable & test: `systemctl start backup.timer` then `systemctl list-timers backup.timer`.

<!-- end_slide -->

# Hands-On 2.3: Socket Activation

1. Create responder script:
```
sudo tee /usr/local/bin/echo-server.sh <<'EOF'
#!/bin/bash
while read line; do echo "You said: $line"; done
EOF
chmod +x /usr/local/bin/echo-server.sh
```
2. `echo.socket`:
```
[Socket]
ListenStream=5555
Accept=yes
Service=echo@.service
```
3. `echo@.service`:
```
[Service]
ExecStart=/usr/local/bin/echo-server.sh
StandardInput=socket
```
4. Test:
```
systemctl enable --now echo.socket
nc localhost 5555
```

<!-- end_slide -->

# Hands-On 2.4: Template Instances

1. Config directories:
```
sudo mkdir -p /etc/colorapp/{red,blue}
```
2. Template service `colorapp@.service`:
```
[Service]
ExecStart=/usr/local/bin/color.sh %i
Restart=always
```
3. Script:
```
sudo tee /usr/local/bin/color.sh <<'EOF'
#!/bin/bash
COLOR=$1
while true; do echo "$COLOR $(date)"; sleep 5; done
EOF
chmod +x /usr/local/bin/color.sh
```
4. Start instances:
```
systemctl start colorapp@red colorapp@blue
journalctl -u colorapp@red -f
```

<!-- end_slide -->

# Troubleshooting Advanced Units

Checklist:
```
systemctl status myapp.service
journalctl -u myapp -b --no-pager
systemd-analyze verify /etc/systemd/system/myapp.service
systemctl show myapp -p FragmentPath -p ActiveState -p ExecMainStatus
systemctl list-dependencies --reverse myapp
systemctl reset-failed myapp
```
Common pitfalls:
- Forgot `daemon-reload`
- Wrong permissions / shebang
- Missing dependencies vs ordering confusion
- Condition line failing silently

<!-- end_slide -->

# Transient & Ephemeral Units

Quick one-off job (with isolation):
```
systemd-run --unit=cache-clean --property=PrivateTmp=yes \
  /usr/local/bin/cleanup-cache.sh
```
Timer + transient service:
```
systemd-run --on-active=5m /usr/bin/echo "Hello later"
```
List:
```
systemctl list-units --state=running | grep run-
```

<!-- end_slide -->

# Performance & Boot Analysis

```
systemd-analyze time
systemd-analyze blame | head
systemd-analyze critical-chain network-online.target
```
Graph dependencies:
```
systemd-analyze dot multi-user.target | dot -Tsvg > graph.svg
```
Identify slow services → optimize or convert to on-demand.

<!-- end_slide -->

# Logging & Filtering Pro Tips

```
journalctl -u myapp -S -2h -p warning
journalctl _SYSTEMD_UNIT=myapp.service _PID=1234
journalctl -t myapp-component
journalctl -u backup.service --since yesterday --output=short-iso
```
Persistent journal check:
```
ls -1 /var/log/journal || echo "Enable persistent storage"
```
Enable persistent:
```
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
```

<!-- end_slide -->

# Exercise: Failure Notification

Add failure hook:
```
[Unit]
Description=Critical Processor
OnFailure=notify@%n.service
```
Notifier:
```
[Unit]
Description=Notify on failure %i

[Service]
Type=oneshot
ExecStart=/usr/local/bin/notify.sh "%i failed on $(hostname)"
```
`notify.sh` placeholder:
```
#!/bin/bash
echo "ALERT: $1" >> /var/log/alerts.log
```

<!-- end_slide -->

# Session 2 Summary

We covered:
- Advanced unit directives (dependencies, conditions)
- Hardening & environment configuration
- Timers, sockets, paths & mounts
- Templates & scaling instances
- Socket & path activation patterns
- Monitoring, watchdog, transient units

Key mindset: Compose primitives → automation platform.

<!-- end_slide -->

# Deliverables & Artifacts

- Cheat sheet (commands + directives)
- Reusable service + timer templates
- Hardened service example
- Backup timer pattern
- Failure notification pattern
- Template multi-instance pattern

Next: assemble into your infra standards repo.

<!-- end_slide -->

# Further Advanced Topics (Explore Later)

- `systemd-nspawn` lightweight containers
- Portable services
- Networkd & resolved integration
- Homed & user units
- Cgroup resource controls (`MemoryMax=`, `CPUQuota=`)
- `sd_notify` advanced readiness signaling

Keep experimenting incrementally.

<!-- end_slide -->

# Resources (Advanced)

- systemd.unit(5) / systemd.service(5)
- systemd.timer(5) / systemd.socket(5)
- systemd.security(7)
- `man systemd.directives`
- Arch Wiki: Advanced systemd
- Poettering blog: design rationale

Practice + repetition → mastery.

<!-- end_slide -->

# Thank You (Session 2)

Experiment, harden, automate.

![systemd_cat](img/systemd_cat.jpg)

<!-- end_slide -->

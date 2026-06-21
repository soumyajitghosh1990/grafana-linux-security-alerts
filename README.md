# Linux Server Authentication Auditing & Grafana Alerting Engine

A production-ready blueprint to scale dynamic SSH Brute-Force and Credential Stuffing alerting across 1,000+ Linux hosts (supporting Ubuntu 24 and Rocky Linux 9 log families) using Promtail, Loki, and Grafana v11.

## 🏗️ Architectural Flowchart

This setup features zero-hardcoding on individual host machines. Instead, systemd dynamically calculates runtime variables at execution and ships metrics upstream:

```text
[ Attacker Terminal ]
        │
        ▼ (Generates Failed SSH Log Entry)
┌────────────────────────────────────────────────────────┐
│ Target Linux Host (Ubuntu 24 / Rocky 9 Fleet Node)     │
│                                                        │
│  1. /var/log/auth.log OR /var/log/secure               │
│        │                                               │
│        ▼ (Watched continuously by daemon)              │
│  2. Promtail Daemon                                    │
│        │─── [Injected dynamically at startup by systemd]
│        │    ├── server_ip = $(hostname -I)             │
│        │    └── host      = $(hostname)                │
└────────┬───────────────────────────────────────────────┘
         │
         ▼ (Ships unified stream with metrics attached)
┌────────────────────────────────────────────────────────┐
│ Loki Log Aggregator Server                             │
└────────┬───────────────────────────────────────────────┘
         │
         ▼ (Evaluates rules every 1 minute)
┌────────────────────────────────────────────────────────┐
│ Grafana v11 Alerting Engine                            │
│                                                        │
│  LogQL Query Pipeline:                                 │
│  sum by (host, server_ip, client_ip) (                 │
│    count_over_time({job="varlogs"} |= "Failed password"│
│    | regexp "from (?P<client_ip>[0-9\\.]+) port")[5m]  │
│  ) > 3                                                 │
└────────┬───────────────────────────────────────────────┘
         │
         ▼ (Routes cleanly; filters out phantom No-Data noises)
┌────────────────────────────────────────────────────────┐
│ Contact Point Notification (Email Inbox)              │
│                                                        │
│  - Targets identified cleanly without static configs!  │
└────────────────────────────────────────────────────────┘
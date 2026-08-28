# Azure GCP — Secure Hybrid Application Platform

## Overview

This project demonstrates a secure hybrid application architecture connecting **Google Cloud Platform (GCP)** and **Microsoft Azure**.

The platform combines infrastructure, networking, security, application delivery, observability, and private cross-cloud database connectivity into one cohesive engineering demonstration.

It is presented as a consolidated architectural implementation and should not be interpreted as a statement about a specific number of years of professional experience.

## Architecture

```text
Administrator
     |
     | IAP / SSH
     v
+---------------------------+
| GCP VPC                   |
| 10.20.0.0/16              |
|                           |
|  Compute Engine VM        |
|  10.20.0.3                |
|       |                   |
|   Nginx :80               |
|       |                   |
|   Gunicorn :8080          |
|       |                   |
|   CNT Web Application     |
+-----------+---------------+
            |
            | IPsec VPN
            v
+---------------------------+
| Azure                     |
| VNet 10.10.0.0/16        |
|                           |
| SQL Private Endpoint      |
| 10.10.3.4:1433            |
|                           |
| Azure SQL Database        |
+---------------------------+
```

## Project Narrative

For the complete engineering story, architecture, validation, security model, current status, and remaining work:

- [Project Narrative](docs/PROJECT_NARRATIVE.md)

## Supporting Evidence

The repository contains sanitized evidence archives corresponding to the validated implementation:

- [Evidence Batch 01 — 16 images](evidence/screenshots/CNT_WebApp_Evidence_Batch_01.zip)
- [Evidence Batch 02 — 16 images](evidence/screenshots/CNT_WebApp_Evidence_Batch_02.zip)

The evidence is intentionally grouped into two ZIP archives so the repository remains readable while retaining the supporting screenshots.

## Key Verified Results

| Component | Verified Result |
|---|---|
| GCP VM | `cnt-prod-vm-01`, RUNNING |
| GCP private IP | `10.20.0.3` |
| GCP VPC | `cnt-prod-vpc` |
| VM external IP | None returned |
| Nginx | Active / enabled |
| Gunicorn | `127.0.0.1:8080` |
| Application | HTTP 200 / healthy |
| Azure VNet | `10.10.0.0/16` |
| Azure SQL Private Endpoint | `10.10.3.4` |
| VPN | ESTABLISHED / Connected |
| Azure SQL TCP 1433 | Connected |
| IAP SSH | Restricted |
| Public HTTP/HTTPS firewall exposure | None verified |
| Secret Manager | Configured |
| Security logging | Configured |
| Terraform | Plan/apply verified with no unexpected changes |

## Security

Never commit real:

- passwords
- VPN shared keys
- API tokens
- service-account private keys
- connection strings containing credentials
- other secret material

Evidence should be sanitized before publication.

## Repository Evidence Layout

```text
.
├── README.md
├── docs/
│   └── PROJECT_NARRATIVE.md
└── evidence/
    └── screenshots/
        ├── CNT_WebApp_Evidence_Batch_01.zip
        └── CNT_WebApp_Evidence_Batch_02.zip
```

## Project Status

The core hybrid networking and private Azure SQL connectivity have been verified.

The broader DR roadmap still contains continuation items such as Cloud Run/Cloud SQL DR workloads, database synchronization, full monitoring/alerting, RPO/RTO testing, failover/recovery exercises, and final operational documentation.

## Engineering Concepts Demonstrated

- Hybrid cloud architecture
- GCP Compute Engine administration
- Azure SQL integration
- Private IP networking
- Site-to-site VPN connectivity
- VPC/VNet design
- Firewall policy
- Identity-aware administrative access
- Nginx reverse proxying
- Gunicorn application serving
- systemd service management
- Process and port troubleshooting
- Secret management
- Centralized audit logging
- Infrastructure verification
- End-to-end connectivity testing
- Terraform infrastructure management
- Operational troubleshooting and recovery

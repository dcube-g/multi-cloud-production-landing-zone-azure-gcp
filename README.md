# Azure Google — Secure Hybrid Application Platform

## Overview

This project demonstrates a secure hybrid application architecture connecting **Google Cloud Platform (GCP)** and **Microsoft Azure**.

The concept brings together infrastructure, networking, security, application delivery, observability, and cross-cloud database connectivity into one cohesive platform.

It is intended to demonstrate **practical infrastructure and cloud engineering knowledge developed and consolidated across multiple projects into a single architectural concept**. It should not be interpreted as a statement of a specific number of years of professional experience.

## Architecture

The platform consists of:

- **GCP Compute Engine** hosting the CNT Web Application
- **Debian 12** virtual machine
- **Nginx** as the HTTP reverse proxy
- **Gunicorn** serving the Python application
- **Azure SQL Database** as the application database
- **GCP VPC** with a dedicated subnet
- **Site-to-site VPN** between GCP and Azure
- **Private Azure SQL connectivity**
- **Google Cloud IAP** for administrative SSH access
- **VPC firewall controls**
- **Google Cloud Secret Manager**
- **Cloud Logging security bucket and audit-log sink**

### Logical flow

```text
Administrator
     |
     | IAP / SSH
     v
+---------------------------+
| GCP VPC                   |
|                           |
|  +---------------------+  |
|  | Compute Engine VM   |  |
|  | 10.20.0.3           |  |
|  |                     |  |
|  | Nginx :80           |  |
|  |      |              |  |
|  | Gunicorn :8080      |  |
|  |      |              |  |
|  | CNT Web Application  |  |
|  +----------+----------+  |
|             |             |
+-------------|-------------+
              |
              | VPN
              v
+---------------------------+
| Azure                     |
|                           |
| Private Endpoint / SQL    |
| 10.10.3.4:1433           |
|                           |
| Azure SQL Database        |
+---------------------------+
```

## Application

The application exposes a health-oriented JSON response confirming the platform components involved:

```json
{
  "application": "CNT Web Application",
  "database": "Azure SQL",
  "environment": "portfolio-test",
  "platform": "GCP Compute Engine",
  "status": "healthy"
}
```

The application is served through Nginx and directly by Gunicorn on the VM.

## Verified Application Stack

### Systemd

The application service is managed by:

```text
cnt-webapp.service
```

Verified state:

```text
active
enabled
```

This means the application is running under systemd and configured to start automatically.

### Nginx

Verified state:

```text
active
enabled
```

Nginx listens on:

```text
0.0.0.0:80
```

### Gunicorn

Gunicorn runs the application on the loopback interface:

```text
127.0.0.1:8080
```

Verified process structure:

```text
gunicorn --workers 2 --bind 127.0.0.1:8080 app.main:app
```

The final verification showed the Gunicorn master process and two workers.

## Application Verification

The final end-to-end verification was performed directly on:

```text
cnt-prod-vm-01
```

### Nginx

```text
HTTP/1.1 200 OK
Server: nginx/1.22.1
Content-Type: application/json
```

Application response:

```json
{
  "application": "CNT Web Application",
  "database": "Azure SQL",
  "environment": "portfolio-test",
  "platform": "GCP Compute Engine",
  "status": "healthy"
}
```

### Gunicorn

```text
HTTP/1.1 200 OK
Server: gunicorn
Content-Type: application/json
```

The same healthy application response was returned directly from Gunicorn.

## Hybrid Network Verification

The GCP VM uses the private address:

```text
10.20.0.3
```

The Azure SQL private address resolved to:

```text
10.10.3.4
```

The route from the GCP VM was verified as:

```text
10.10.3.4 via 10.20.0.1 dev ens4 src 10.20.0.3
```

TCP connectivity to Azure SQL was verified successfully:

```text
TCP 1433 CONNECTED
```

DNS verification also confirmed the Azure SQL hostname resolves to the private endpoint:

```text
10.10.3.4 cnt-webapp-prod-cin-sql-001.privatelink.database.windows.net
```

This provides evidence that the application host can reach the Azure SQL private address through the hybrid network path.

## VPN

The GCP-to-Azure VPN tunnel was verified as:

```text
NAME: cnt-prod-to-azure-tunnel-0
REGION: asia-south1
STATUS: ESTABLISHED
PEER_ADDRESS: 4.213.213.25
```

The tunnel provides the private network path between the GCP and Azure environments.

## Network Security

### No External IP

The VM has a private network address:

```text
10.20.0.3
```

The external/NAT IP query returned no value, providing evidence that the VM does not have an assigned external IP.

### IAP SSH Access

Administrative SSH access is restricted through the IAP firewall rule:

```text
cnt-prod-vpc-allow-iap-ssh
```

The rule permits:

```text
TCP 22
Source: 35.235.240.0/20
Target tag: ssh
```

The VM has the corresponding tag:

```text
ssh
```

This supports administrative access without exposing SSH broadly to the public internet.

### Internal Firewall

The internal rule is:

```text
cnt-prod-vpc-allow-internal
```

Source range:

```text
10.20.0.0/24
```

Allowed protocols:

```text
ICMP
UDP
TCP
```

No application-specific public HTTP, HTTPS, or port 8080 ingress rule was present in the verified VPC firewall rules.

## Service Exposure Model

The application is intentionally structured as:

```text
External / controlled entry
        |
        v
      Nginx :80
        |
        v
 Gunicorn 127.0.0.1:8080
        |
        v
   Python application
```

Gunicorn is bound only to:

```text
127.0.0.1:8080
```

Therefore the application server itself is not directly exposed on the VM's network interface.

Nginx is the HTTP-facing layer.

## Secrets

Google Cloud Secret Manager contains:

```text
my-app-secret
```

Secrets are therefore represented as managed secret material rather than being documented as plaintext configuration values in this README.

## Logging and Security Monitoring

A dedicated Cloud Logging bucket was verified:

```text
cnt-prod-security-logs
```

Location:

```text
asia-south1
```

Retention:

```text
30 days
```

A security sink was also verified:

```text
cnt-prod-security-sink
```

Destination:

```text
logging.googleapis.com/projects/cnt-webapp-prod-gcp-jd01/locations/asia-south1/buckets/cnt-prod-security-logs
```

The verified filter targets Cloud Audit Logs:

```text
logName:"logs/cloudaudit.googleapis.com"
```

This provides a foundation for retaining and reviewing security-relevant audit activity.

## Infrastructure Evidence

### GCP Project

```text
cnt-webapp-prod-gcp-jd01
```

### Compute Engine

```text
VM: cnt-prod-vm-01
Zone: asia-south1-a
Private IP: 10.20.0.3
Status: RUNNING
OS: Debian GNU/Linux 12 (bookworm)
```

### VPC

```text
VPC: cnt-prod-vpc
Auto-created subnetworks: false
Routing mode: REGIONAL
```

### Subnet

```text
cnt-prod-subnet
```

The VM network interface is attached to this subnet.

## Operational Issue Identified and Resolved

During service validation, an orphaned Python process was found holding port 8080:

```text
PID 3816
PPID 1
python3 app/main.py
```

This process remained after the application service was stopped and caused Gunicorn to report:

```text
Connection in use: ('127.0.0.1', 8080)
Address already in use
```

The orphaned process was terminated and the application service was restarted.

After cleanup, the expected service model was restored:

```text
cnt-webapp.service
        |
        +-- Gunicorn master
        |
        +-- Gunicorn worker
        |
        +-- Gunicorn worker
```

The application then successfully listened on:

```text
127.0.0.1:8080
```

This incident demonstrated the importance of identifying the actual process owning a listening socket rather than assuming that stopping a systemd unit has removed every application process.

## Verification Summary

Final verified state:

| Component | Result |
|---|---|
| VM | RUNNING |
| VM external IP | None returned |
| Debian | 12 |
| VPC | `cnt-prod-vpc` |
| VM private IP | `10.20.0.3` |
| Nginx | active / enabled |
| CNT systemd service | active / enabled |
| Gunicorn | running |
| Nginx port | `80` |
| Gunicorn port | `127.0.0.1:8080` |
| Application via Nginx | HTTP 200 / healthy |
| Application via Gunicorn | HTTP 200 / healthy |
| Azure SQL DNS | private `10.10.3.4` |
| Azure SQL route | verified |
| Azure SQL TCP 1433 | connected |
| VPN | ESTABLISHED |
| IAP SSH firewall | restricted to IAP range |
| Public HTTP/HTTPS firewall rule | none verified |
| Secret Manager | `my-app-secret` |
| Security log bucket | 30-day retention |
| Audit sink | configured |

## Evidence Collection

The implementation was validated through direct infrastructure and application checks, including:

```bash
systemctl is-active cnt-webapp.service
systemctl is-enabled cnt-webapp.service

systemctl is-active nginx
systemctl is-enabled nginx

ss -lntp

curl -fsS http://127.0.0.1/
curl -fsS http://127.0.0.1:8080/

getent hosts cnt-webapp-prod-cin-sql-001.database.windows.net

ip route get 10.10.3.4

timeout 5 bash -c '</dev/tcp/10.10.3.4/1433'

gcloud compute vpn-tunnels list

gcloud compute firewall-rules list

gcloud compute instances describe cnt-prod-vm-01

gcloud secrets list

gcloud logging buckets describe cnt-prod-security-logs

gcloud logging sinks describe cnt-prod-security-sink
```

## Key Engineering Concepts Demonstrated

This project consolidates the following engineering concepts into one platform:

- Hybrid cloud architecture
- GCP Compute Engine administration
- Azure SQL integration
- Private IP networking
- Site-to-site VPN connectivity
- VPC design
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
- Operational troubleshooting and recovery

## Important Architectural Principle

The central idea is not simply that multiple cloud products were deployed.

The value of the project is the **integration of the individual engineering concerns into one coherent system**:

```text
Infrastructure
      +
Networking
      +
Security
      +
Application delivery
      +
Database connectivity
      +
Observability
      +
Operational troubleshooting
      =
Integrated hybrid cloud platform
```

The implementation is therefore presented as a **single consolidated concept built from knowledge and patterns developed through multiple projects**, rather than as a claim of a particular duration of professional experience.

## Project Status

**Implementation verified successfully.**

The final evidence confirms:

1. The GCP VM is running without an external IP.
2. Nginx and the application service are enabled and active.
3. Gunicorn is correctly bound to localhost on port 8080.
4. The application returns a healthy response through both Nginx and Gunicorn.
5. The GCP-to-Azure VPN tunnel is established.
6. Azure SQL resolves to its private address.
7. The GCP VM has a valid route to the Azure SQL private address.
8. TCP connectivity to Azure SQL port 1433 succeeds.
9. Administrative SSH access is restricted through IAP.
10. No public HTTP/HTTPS firewall exposure was identified in the verified VPC rules.
11. Secret Manager and centralized audit logging are configured.
12. An application port conflict caused by an orphaned process was identified, cleaned up, and resolved.

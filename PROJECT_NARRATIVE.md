# CNT WebApp — Multi-Cloud Project Narrative

## Executive Summary

This project demonstrates a secure hybrid application platform spanning Microsoft Azure and Google Cloud Platform (GCP). Azure is treated as the primary production environment and GCP as the secondary/DR environment.

The implementation brings together private networking, cross-cloud IPsec VPN connectivity, private DNS, Azure SQL Private Endpoint access from GCP, application hosting, infrastructure verification, security controls, logging, and operational troubleshooting.

The project is presented as a consolidated cloud-engineering implementation rather than a claim about a specific amount of professional experience.

## Architecture

```text
                 AZURE — PRIMARY
                 Central India
                      |
          +-----------+-----------+
          |                       |
      Azure VNet              Azure SQL
     10.10.0.0/16                |
          |                 Private Endpoint
     VPN Gateway              10.10.3.4
          |
          | IPsec VPN
          |
       GCP HA VPN
          |
       GCP VPC
      10.20.0.0/16
          |
    Compute Engine VM
       10.20.0.3
          |
       Nginx :80
          |
  Gunicorn 127.0.0.1:8080
          |
    CNT Web Application
```

The documented target architecture also includes Azure App Service/AKS and a GCP DR pattern using Cloud Run and Cloud SQL. Those DR workload components remain continuation items rather than being represented as completed production resources.

## Verified Implementation

### GCP application host

- Project: `cnt-webapp-prod-gcp-jd01`
- VPC: `cnt-prod-vpc`
- VM: `cnt-prod-vm-01`
- Zone: `asia-south1-a`
- Private IP: `10.20.0.3`
- OS: Debian GNU/Linux 12
- No external IP was returned for the VM.

### Application stack

The application is managed with systemd:

- Service: `cnt-webapp.service`
- State: active and enabled
- Nginx: active and enabled
- Nginx: `0.0.0.0:80`
- Gunicorn: `127.0.0.1:8080`

The application returned HTTP 200 and a healthy JSON response through both Nginx and Gunicorn.

## Azure Integration

The primary Azure network uses:

- VNet: `cnt-webapp-prod-cin-vnet-001`
- CIDR: `10.10.0.0/16`
- Web subnet: `10.10.1.0/24`
- API/AKS subnet: `10.10.2.0/24`
- Data/private-endpoint subnet: `10.10.3.0/24`
- DNS Resolver subnet: `10.10.4.0/28`
- Gateway subnet: `10.10.255.0/27`

Azure SQL:

- Server: `cnt-webapp-prod-cin-sql-001`
- Database: `cnt-webapp-prod-cin-sqldb-orders-001`
- Private Endpoint: `cnt-webapp-prod-cin-sql-pe-001`
- Private IP: `10.10.3.4`
- Minimum TLS: 1.2

## Hybrid Connectivity

The GCP-to-Azure VPN path was validated end to end.

GCP:

- HA VPN tunnel: `cnt-prod-to-azure-tunnel-0`
- Region: `asia-south1`
- Tunnel status: `ESTABLISHED`
- Route to Azure: `10.10.0.0/16`

Azure:

- VPN Gateway: `cnt-webapp-prod-cin-vpng-001`
- Gateway type: RouteBased
- SKU: `VpnGw2AZ`
- Azure-side public/tunnel address used by the connection: `4.213.213.25`

The networks are intentionally non-overlapping:

```text
Azure: 10.10.0.0/16
GCP:   10.20.0.0/16
```

## Private DNS and SQL Connectivity

The most important connectivity proof is the complete private path from the GCP VM to Azure SQL.

The Azure SQL hostname resolves through the private DNS path to:

```text
10.10.3.4
```

The GCP VM route was verified:

```text
10.10.3.4 via 10.20.0.1 dev ens4 src 10.20.0.3
```

TCP port 1433 was successfully reached.

This demonstrates:

1. Cross-cloud VPN routing is functioning.
2. Private DNS resolution is functioning.
3. Azure SQL is reachable through its Private Endpoint.
4. The application host can reach the database without requiring a public SQL endpoint.

## Security Controls

The verified GCP security model includes:

- No external IP on the application VM.
- IAP-based SSH administration.
- SSH restricted through the IAP source range.
- Internal firewall rules scoped to the GCP subnet.
- No verified public HTTP/HTTPS firewall rule for the application.
- Gunicorn bound to localhost rather than the VM network interface.
- Google Cloud Secret Manager used for managed secret material.
- Centralized Cloud Logging security bucket and audit sink.

The repository must never contain real credentials, VPN shared keys, passwords, tokens, private keys, or plaintext secrets.

## Azure Security and Policy

Azure policy enforcement was encountered during the build and incorporated into the deployment workflow.

The `Environment=prod` requirement was accounted for on resources that support tags.

The project also demonstrates an important operational principle: existing cloud resources should be inspected and reused rather than blindly recreated. Existing Azure VNet, VPN Gateway, SQL, and private DNS resources were treated as existing infrastructure where appropriate.

## Terraform / Infrastructure as Code

Terraform is the intended infrastructure-management layer.

The validation workflow is:

```bash
terraform init
terraform fmt -check
terraform validate
terraform plan
```

A verified Terraform run reported:

```text
No changes. Your infrastructure matches the configuration.
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

The project documentation recommends importing existing Azure resources into Terraform rather than recreating them.

## Operational Troubleshooting

During application validation, an orphaned Python process was found holding port 8080 after the application service had been stopped. This caused Gunicorn to report an address-in-use condition.

The process was identified, removed, and the service was restarted. The expected systemd → Gunicorn master → workers model was then restored.

This is included as evidence of practical troubleshooting rather than hiding the incident.

## Current Status

### Completed / verified

- Azure production networking foundation
- Azure VNet and subnet design
- Azure VPN Gateway
- GCP VPC
- GCP Compute Engine VM
- GCP HA VPN
- Azure ↔ GCP IPsec connectivity
- GCP route to Azure
- Azure Local Network Gateway
- Azure VPN connection
- Azure SQL
- Azure SQL Private Endpoint
- Azure Private DNS
- Azure DNS Resolver path
- GCP Cloud DNS forwarding
- GCP → Azure private DNS resolution
- GCP → Azure SQL TCP/1433 connectivity
- Application health checks
- IAP administrative access
- GCP firewall validation
- Secret Manager foundation
- Centralized audit logging
- Terraform plan/apply verification

### Remaining / continuation work

The full production/DR roadmap still includes:

- Additional Terraform modules and state hardening
- Final Azure policy/compliance review
- Least-privilege IAM cleanup
- Full GCP monitoring and alerting
- Final cost governance
- Cloud Run DR application
- Cloud SQL DR database
- Documented database synchronization
- RPO/RTO definition
- Controlled failover and recovery testing
- Final architecture diagram and resource inventory
- Incident/RCA and change-management exercise

These items should be described as remaining work and not presented as completed evidence.

## Supporting Evidence

The repository should contain the two sanitized evidence archives:

- `CNT_WebApp_Evidence_Batch_01.zip` — 16 images
- `CNT_WebApp_Evidence_Batch_02.zip` — 16 images

The evidence archives are intended to support the narrative without cluttering the repository with dozens of individual screenshots.

## Evidence Handling and Security

Evidence was intentionally sanitized before repository publication.

Do not upload:

- credentials
- passwords
- VPN shared secrets
- service-account private keys
- API tokens
- connection strings containing credentials
- screenshots exposing secret values
- unnecessary tenant/subscription identifiers when they are not needed for proof

The objective is to provide enough evidence for independent review while keeping the repository safe to publish.

## Reviewer Summary

The strongest technical proof in this project is the end-to-end chain:

```text
GCP VM
10.20.0.3
   |
   | GCP HA VPN
   v
Azure VNet
10.10.0.0/16
   |
   +--> Azure DNS Resolver
   |
   +--> Private DNS
   |
   v
Azure SQL Private Endpoint
10.10.3.4:1433
```

Combined with the application health check, private routing, firewall validation, IAP access model, Terraform verification, and evidence archives, this provides a concise demonstration of the project's principal engineering outcomes.

# Architecture & Major Evidence

This document is the visual companion to the project README and
`PROJECT_NARRATIVE.md`. It highlights the major architectural and
validation evidence rather than reproducing all 32 supporting
screenshots.

## 1. Architecture overview

![Azure architecture](images/01-azure-architecture.png)

The implementation uses Azure as the primary network/database side and
GCP as the application-host side. The verified private ranges are
non-overlapping:

-   Azure: `10.10.0.0/16`
-   GCP: `10.20.0.0/16`

The documented private path is:

``` text
GCP VM 10.20.0.3
      |
      | GCP HA VPN / IPsec
      v
Azure VNet 10.10.0.0/16
      |
      +--> Azure DNS Resolver / Private DNS
      |
      v
Azure SQL Private Endpoint
10.10.3.4:1433
```

## 2. Azure VPN / hybrid connectivity

![Azure VPN](images/02-azure-vpn.png)

This screenshot provides the Azure-side VPN gateway evidence supporting
the cross-cloud private connectivity design.

The verified GCP tunnel is `cnt-prod-to-azure-tunnel-0`, with status
`ESTABLISHED`.

## 3. GCP network and subnet

![GCP network subnet](images/03-gcp-network-subnet.png)

The GCP production VPC and subnet evidence establishes the application
network boundary used by the Compute Engine VM.

The application VM private address is `10.20.0.3`.

## 4. Firewall security boundary

![GCP firewall](images/04-gcp-firewall.png)

The firewall evidence demonstrates the controlled ingress model. SSH
administration is handled through IAP, while the application itself is
not exposed through a broad public HTTP/HTTPS or port-8080 firewall
rule.

## 5. IAP-based administration

![IAP SSH](images/05-iap-ssh-security.png)

The IAP evidence supports the administrative access model. The verified
SSH rule permits TCP/22 from the Google IAP source range rather than
exposing SSH broadly.

## 6. Terraform plan

![Terraform plan](images/06-terraform-plan.png)

Terraform validation reported:

``` text
No changes. Your infrastructure matches the configuration.
```

This is an important IaC verification point because the deployed
infrastructure matched the configuration at the time of the check.

## 7. Terraform apply

![Terraform apply](images/07-terraform-apply.png)

The apply verification reported:

``` text
Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

This demonstrates a clean, non-destructive Terraform state at the time
of validation.

## 8. Azure SQL

![Azure SQL overview](images/08-azure-sql-overview.png)

Azure SQL is the private database target for the hybrid connectivity
demonstration. The Private Endpoint address used in the verified path is
`10.10.3.4`.

## 9. Application verification

![Application verification](images/09-application-verification.png)

The application returned a healthy HTTP response. The verified
application stack is:

``` text
Nginx :80
   |
Gunicorn 127.0.0.1:8080
   |
CNT Web Application
```

## 10. GCP VM verification

![GCP VM verification](images/10-gcp-vm-verification.png)

The GCP Compute Engine VM is the application host used for the
end-to-end private connectivity test.

Verified identity:

-   VM: `cnt-prod-vm-01`
-   Zone: `asia-south1-a`
-   Private IP: `10.20.0.3`
-   OS: Debian GNU/Linux 12
-   No external IP returned

## 11. End-to-end proof

The strongest technical evidence is the complete chain:

``` text
Application host
     |
     v
GCP VM 10.20.0.3
     |
     | route
     v
Azure private network 10.10.0.0/16
     |
     | private DNS
     v
Azure SQL Private Endpoint 10.10.3.4
     |
     | TCP/1433
     v
CONNECTED
```

This proves more than simply showing that a VPN tunnel exists: the
route, DNS resolution, private endpoint address, and database TCP port
were validated from the GCP VM.

## 12. Evidence publication guidance

Only the major screenshots are embedded here. The full supporting
evidence remains in the two evidence batches.

Do not publish passwords, VPN shared keys, API tokens, private keys,
connection strings containing credentials, or other secret material.

### Suggested repository layout

``` text
.
├── README.md
├── docs/
│   ├── PROJECT_NARRATIVE.md
│   └── architecture/
│       ├── ARCHITECTURE_EVIDENCE.md
│       └── images/
│           ├── 01-azure-architecture.png
│           ├── 02-azure-vpn.png
│           ├── 03-gcp-network-subnet.png
│           ├── 04-gcp-firewall.png
│           ├── 05-iap-ssh-security.png
│           ├── 06-terraform-plan.png
│           ├── 07-terraform-apply.png
│           ├── 08-azure-sql-overview.png
│           ├── 09-application-verification.png
│           └── 10-gcp-vm-verification.png
└── evidence/
    └── screenshots/
        ├── CNT_WebApp_Evidence_Batch_01.zip
        └── CNT_WebApp_Evidence_Batch_02.zip
```

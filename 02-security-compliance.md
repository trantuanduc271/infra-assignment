# Plugilo Infrastructure

This document outlines the security and compliance architecture design for Plugilo's global SaaS platform.

## Part 2 – Security & Compliance

### 1. Context and Requirements

**Platform:**
- 10–15 million users
- Microservices architecture
- Backend services in Node.js / Python / Java
- Databases: PostgreSQL, MongoDB, Elasticsearch, Redis
- AI workloads (GPU optional; no deep ML required)
- Web & mobile clients (Web, iOS, Android)
- Deployed on Azure (primary) with multi-cloud connectivity to AWS and GCP

**Key Security Requirements:**
- GDPR compliance and data protection
- Zero Trust security model
- Defense in depth
- Least privilege access
- Comprehensive audit logging
- Incident response readiness

### 1.1 Compliance Auditing

**What is Vanta:**
Vanta is a compliance automation platform that continuously monitors security controls and automates evidence collection for compliance frameworks (GDPR, ISO 27001, SOC 2, HIPAA, PCI DSS). It integrates with cloud providers, development tools, and security systems to provide real-time compliance status, automated audit trails, and remediation workflows.

**Decision: Vanta for Compliance Auditing and Continuous Monitoring**

**Rationale:**
- **Compliance Frameworks**: Automated compliance monitoring for GDPR, ISO 27001, SOC 2, HIPAA, PCI DSS
- **Cloud Agnostic**: Compatible with Azure, AWS, and GCP, supporting multi-cloud compliance monitoring
- **Continuous Monitoring**: Real-time compliance status tracking, automated evidence collection, and gap identification
- **Automated Remediation**: Integrates with ticket systems (Jira, GitHub Issues) to automatically create tickets for DevOps teams to fix compliance gaps until they pass
- **Integration**: Integrates with Azure services, GitHub, CI/CD pipelines, and security tools for comprehensive coverage
- **Audit Readiness**: Automated audit trail generation, compliance reports, and evidence documentation

---

## 2. IAM / RBAC Strategy

### 2.1 Identity and Access Management

**Decision: Microsoft Entra ID (Azure AD) as Primary Identity Provider**

**Rationale:** Centralized identity management, Azure service integration (Managed Identities, RBAC), MFA and conditional access, federation capabilities.

**Key Components:**
- **Managed Identities**: System/user-assigned identities for Azure resources (AKS, Functions, VMs) to access Azure services without credentials
- **Service Principals**: CI/CD pipelines and automated deployments
- **Conditional Access Policies**: Location-based, device compliance, risk-based access controls

### 2.2 Role-Based Access Control (RBAC)

**Azure RBAC Hierarchy:** Management Group (platform-wide) → Subscription (environment-specific) → Resource Group (application/service) → Resource (granular)

**Role Definitions:** Infrastructure Admin, Application Developer (dev/staging access, read-only production), Security Admin, Read-Only Auditor, DevOps Engineer

**Just-In-Time (JIT) Access:** PIM for elevated permissions, time-bound access with approval workflows, automatic revocation

---

## 3. Network Security Boundaries

### 3.1 Public vs Private Network Segmentation

**Network Zones:** Public (Internet-facing: Front Door), DMZ (Application Gateway, APIM with private endpoints), Private (Application workloads, databases), Management (Bastion, monitoring)

**Security Controls:** Azure Firewall (centralized security with DPI, IPS, FQDN/URL filtering, Threat Intelligence), NSGs (micro-segmentation), Private Endpoints (all PaaS services), Private DNS Zones

### 3.2 Firewall Selection

**Decision: Azure Firewall vs NVA (Network Virtual Appliance)**

| Aspect | Azure Firewall | NVA (Checkpoint/Palo Alto/Fortigate) |
|--------|----------------|--------------------------------------|
| Deep Packet Inspection (DPI) | Basic | Advanced - full packet inspection |
| Intrusion Prevention (IPS) | Basic | Advanced - signature-based and behavioral |
| Application Layer Filtering | Limited | Advanced - application-aware policies |
| Threat Intelligence | Azure-native feeds | Vendor-specific + third-party feeds |
| SSL/TLS Inspection | Limited | Full SSL decryption and inspection |
| URL Filtering | Basic | Advanced - category-based filtering |
| Cost | Lower - pay-per-use | Higher - licensing + compute |
| Management | Native Azure integration | Vendor-specific management console |
| Vendor Lock-in | Azure-native | Vendor-specific (but portable) |

**Decision: Azure Firewall** selected for native Azure integration, cost efficiency, and sufficient security capabilities for current requirements. This choice reduces operational complexity (no separate vendor management console, unified Azure management) and minimizes vendor dependency (native Azure service vs third-party NVA licensing and support).

### 3.3 NSG vs Azure Firewall

| Aspect | NSG | Azure Firewall |
|--------|-----|----------------|
| Stateful Filtering | Yes | Yes |
| Application Layer Filtering | No - port and IP only | Yes - FQDN, URL filtering |
| Dynamic IP Handling | Limited | Yes - FQDN-based rules |
| Threat Intelligence | No | Yes - Azure-native feeds |
| Centralized Management | No - per-subnet | Yes - centralized policies |
| Cost | Included | Additional cost |
| Complexity | Lower - simpler rules | Higher - more features |

**Strategy**: NSGs for micro-segmentation within subnets, Azure Firewall for centralized network security and advanced filtering.

---

## 4. Secrets Management

### 4.1 Secrets Management Strategy

**Decision: External Secrets Operator (ESOP) for Kubernetes Secrets**

**Rationale:** Cloud-agnostic (supports Azure Key Vault, AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault), unified interface, better portability than Azure-native CSI.

**Secret Management:**
- **CI/CD Pipeline**: GitHub Secrets for pipeline credentials, API keys, deployment tokens
- **Kubernetes Runtime**: ESOP syncs secrets from Azure Key Vault to Kubernetes, Key Vault (Premium tier, HSM-backed keys, geo-redundant, private endpoints), Kubernetes-Reflector for cross-namespace distribution

### 4.2 Secret Rotation

**Automatic:** Azure Key Vault automatic rotation, ESOP syncs to Kubernetes, Reloader reloads pods

**Manual:** Key Vault versioning, rollback capability, audit trail for all access and modifications

---

## 5. Data Encryption

Data encryption in Azure is divided into two types: encryption in-transit and encryption at-rest.

### 5.1 Encryption in Transit

Encryption in-transit encrypts data during transmission using SSL/TLS protocols and IPSEC (site-to-site VPN). All Azure services provide endpoints with encryption enabled by default or offer options to enable encryption:

| Service Type | Encryption Type |
|--------------|----------------|
| Azure PaaS services (Service Bus, Event Grid, etc.) | HTTPS enabled by default |
| Azure Database services (PostgreSQL, MongoDB, etc.) | Always TLS enabled, users can choose encryption based on client configuration |
| Application Gateway | Integrated with Azure Key Vault or Azure App Service Certificates for SSL certificate management and HTTPS encryption |
| API Management | TLS termination and mTLS for service-to-service communication |
| VPN Gateway (site-to-site) | IPSEC encryption for connecting on-premises networks to Azure VNet |

Typically, traffic within VNet is relatively secure and encryption is not required, however users can enable encryption features provided by services if needed. We always encrypt traffic to the Internet and enable HTTP to HTTPS redirect to ensure users always use HTTPS protocol.

**Additional Encryption:**
- **Istio Service Mesh**: mTLS for inter-service communication within AKS clusters
- **TLS 1.3**: Minimum version for all external-facing services

**Certificate Management:** Azure Key Vault (certificate storage and automatic renewal), Let's Encrypt/Azure App Service Certificates (public domains), Private CA (internal services)

### 5.2 Encryption at Rest

Encryption at-rest is the ability to encrypt data at the storage location. All Azure storage-related services provide server-side encryption at-rest, and the encryption/decryption process is completely automatic. Azure encrypts data before storing on disk and decrypts before returning data to users (if users have decryption permissions). Keys used for encryption and decryption are managed by Azure Key Vault. Services used include:

| Service Type | Encryption Type |
|--------------|----------------|
| Azure Storage (Blob, File, Queue, Table) | Default not encrypted, we enable encryption in all cases. Typically use Azure-managed encryption or Key Vault (customer-managed keys) |
| Azure Database for PostgreSQL | Disk encryption with Key Vault (customer-managed keys) |
| Azure Cosmos DB (MongoDB) | Encryption at rest enabled by default |
| Azure Cache for Redis | Premium tier with encryption at rest |
| Azure Disks (Managed Disks) | Encryption with Key Vault (customer-managed keys) |
| Azure Container Registry | Encryption at rest |
| Azure Backup | Encryption at rest |

**Key Management:** Azure Key Vault (HSM-backed keys, Premium tier), automated rotation policies, RBAC and access policies

---

## 6. Environment Isolation

### 6.1 Environment Separation Strategy

**Decision: Subscription-Based Isolation**

**Rationale:** Strong isolation boundaries, independent cost tracking, separate network topologies and security policies, isolated IAM.

**Environment Structure:** Production (highest security controls), Staging (mirrors production), Development (multiple, one per team/feature), UAT

### 6.2 Network Isolation

**VNet Isolation:** Separate VNets per environment, no direct connectivity between environments, hub-and-spoke per environment, private endpoints for all PaaS services

**Access Control:** Production (AVD for compliant access, JIT with approval), Non-Production (Tailscale VPN), Kubernetes Network Policies for pod-to-pod control

### 6.3 Resource Isolation

**Resource Groups:** Logical grouping by application/service, environment-specific tags, separate CI/CD service principals per environment

**Kubernetes Namespaces:** Namespace-based isolation, Network Policies for traffic control, RBAC for access control

---

## 7. Audit Logging

### 7.1 Audit Logging Strategy

**Decision: Centralized Logging with Azure Monitor and Log Analytics**

**Key Components:** Azure Activity Log (subscription-level events), Resource Logs (resource-specific), Key Vault Logging (secret/key access), Azure AD Sign-In Logs (authentication), Kubernetes Audit Logs (API server requests)

### 7.2 Log Retention and Compliance

**Retention:** Security logs (1 year, GDPR), Audit logs (7 years, regulatory), Application logs (90 days), Cost logs (2 years)

**Storage:** Azure Log Analytics Workspace (centralized aggregation), Azure Storage (long-term archival), Immutable WORM storage for audit logs

### 7.3 Log Monitoring and Alerting

**Security Monitoring:** Azure Sentinel (SIEM, event correlation, threat detection), Custom alerts (failed auth, unauthorized access, suspicious activities), Automated compliance reports (GDPR, SOC 2, ISO 27001)

---

## 8. Incident Response Overview

### 8.1 Incident Response Plan

**Incident Classification:** Critical (P0 - production outage, data breach, credential compromise), High (P1 - security vulnerability, service degradation), Medium (P2 - non-critical issues), Low (P3 - minor issues)

**Response Team:** Incident Commander (Infrastructure Lead/on-call), Security Team (analysis/containment), Engineering Team (restoration/remediation), Communication Lead (stakeholder updates)

### 8.2 Incident Response Process

**Process:** Detection → Containment → Eradication → Recovery

- **Detection**: Automated alerts, SIEM, user reports
- **Containment**: Isolate systems, revoke credentials, network segmentation
- **Eradication**: Root cause analysis, patching, hardening
- **Recovery**: Service restoration, monitoring, post-incident review

---

## 9. GDPR and Data Protection Considerations

### 9.1 Data Residency and Location

**Decision: EU Regions (Germany West Central, Germany North) for Data Residency**

**Rationale:** GDPR requires EU/EEA data processing, Germany regions ensure compliance, paired regions provide DR while maintaining data residency.

**Data Processing:** Primary (Germany West Central/Frankfurt), Secondary (Germany North/Berlin for DR, replication within EU), No data transfer outside EU/EEA without consent and safeguards

### 9.2 Data Protection Measures

**Data Minimization:** Collect only necessary data, automatic deletion after retention, user data deletion on request (Right to be Forgotten)

**Data Subject Rights:** Right to Access, Rectification, Erasure, Portability (export in machine-readable format)

**Privacy by Design:** Encryption at rest/in transit, access controls and audit logging, data anonymization for analytics/testing

### 9.3 Data Breach Notification

**Breach Detection:** Automated monitoring for unauthorized access, security alerts, regular audits and penetration testing

**Breach Response:** 72-hour notification to supervisory authority, user notification without undue delay if high risk, detailed breach records

---

## 10. Required Questions

### 10.1 How do you protect production from unauthorized internal access?

**Network Architecture:** Production VNet isolated from non-production (hub-and-spoke topology, no direct connectivity), private endpoints for all PaaS services, Azure Firewall for centralized security inspection, NSGs for micro-segmentation, Kubernetes Network Policies for pod-to-pod control.

**Access Control:** AVD (Azure Virtual Desktop) for production access only (GDPR-compliant, geo-restricted), JIT access (PIM with time-bound access, approval workflows), Conditional Access Policies (location-based, device compliance, risk-based), RBAC with least privilege, regular access reviews.

**Monitoring:** Real-time monitoring of access attempts, alerts for unauthorized access and suspicious activities, immutable audit logs for all production access.

### 10.2 How would you respond in the first 24 hours after a credential leak?

**Response:**

1. **Announce Incident**: Notify security team, management, and affected teams
2. **Revoke Credentials**: Revoke compromised credentials (service principals, API keys, access tokens) via Azure AD and Key Vault
3. **Rotate Secrets**: Rotate exposed secrets (database passwords, API keys, certificates) using Key Vault automatic rotation
4. **Isolate Systems**: Network isolation using NSGs and Azure Firewall rules
5. **Enhanced Monitoring**: Increase logging and monitoring in Azure Monitor and Sentinel
6. **Investigate**: Review access logs (Azure AD, Key Vault, Kubernetes audit logs), assess scope, identify attack vector
7. **Containment**: Update security policies (tighten access controls, conditional access), document timeline
8. **Remediation**: Implement fixes (remove exposed credentials, update training, secret scanning), restore services with enhanced controls, schedule post-mortem

---


# Case Management System High-Level Design (HLD)

## 1. Overview

**Purpose:**
Design a secure, resilient Azure-based Case Management System for agents to update case stakeholders with relevant information and status changes.

**Scope:**
- Agent-facing case update portal
- Stakeholder notification delivery (email/SMS/push)
- Case data storage, search, and audit tracking
- Role-based access for agents, supervisors, and case participants
- UK data residency and enterprise governance

**System Summary:**
A PaaS-first architecture using Azure App Service / Container Apps for application hosting, Azure SQL Database for case data, Azure Communication Services for stakeholder notifications, and Azure AD for identity and access control.

---

## 2. Architecture Summary

**High-level design approach:**
- Use a modular microservice-style architecture with API and UI tiers separated
- Prioritize managed Azure services to minimize operational overhead
- Embed observability and governance in the design
- Support near real-time stakeholder updates via event-driven notifications

**Key architectural patterns:**
- API Gateway / service front-end
- Event-driven messaging for notifications
- Multi-tier security with Azure AD and private networking
- Read-optimized cache for frequently accessed case data
- Zone-redundant database and app deployment for availability

---

## 3. Service Selection & Rationale (Cloud Architect)

**Frontend / Agent Experience**
- `Azure App Service` or `Azure Container Apps` for the agent portal and case management API
- Rationale: PaaS hosting with built-in autoscale, integration with Azure AD, and reduced platform management. Container Apps is preferred if there are multiple microservices or workloads requiring ephemeral scale.

**API / Backend**
- `Azure App Service` / `Azure Container Apps` for REST APIs
- `Azure API Management` optionally for API governance and secure external access if third-party integrations are required
- Rationale: Managed deployment, built-in authentication, and easier versioning.

**Data Storage**
- `Azure SQL Database` (Business Critical or General Purpose with Zone Redundant Configuration)
- `Azure Cache for Redis` for frequently accessed case metadata and dashboard views
- Rationale: Relational data model for cases, updates, assignments, audit history, with managed backups and geo-redundancy.

**Notifications**
- `Azure Communication Services` for email/SMS/chat notifications
- `Azure SignalR Service` for real-time agent dashboard updates and stakeholder web notifications
- Rationale: Unified communications channel and low-latency push capabilities for case updates.

**Identity & Access**
- `Azure Active Directory` for agent authentication and RBAC
- `Azure AD B2B/B2C` if external stakeholders need portal access
- Rationale: Zero Trust identity-first security and centralized access control.

**Observability**
- `Azure Monitor` + `Application Insights` for telemetry, custom metrics, and alerting
- `Azure Log Analytics` for audit and diagnostic logs
- Rationale: Operational excellence and incident response.

**Security / Governance**
- `Azure Policy` for enforced tagging, allowed locations, secure networking, and SQL auditing
- `Azure Key Vault` for any secrets, certificates, and API keys
- Rationale: Policy-driven governance and secret management.

**Integration / Workflow**
- `Azure Service Bus` for reliable notification delivery and case event orchestration
- `Azure Functions` for lightweight event handlers, retry logic, and scheduled jobs
- Rationale: Decoupled processing and resiliency for asynchronous updates.

---

## 4. Well-Architected Assessment

### Security
- Use Azure AD authentication for all user access.
- Protect backend APIs with managed identities and OAuth tokens.
- Store credentials only in `Azure Key Vault`; avoid secrets in app configuration.
- Apply network security with `Private Endpoint` for SQL and Service Bus.
- Use `Azure Policy` to enforce secure transfer and TDE on SQL.

### Reliability
- Deploy App Service / Container Apps across UK South and UK West in a primary/secondary active-passive pattern.
- Enable Zone Redundant Configuration on Azure SQL Database.
- Use `Service Bus` dead-letter queues and retry policies for notification delivery.
- Maintain automated backups and point-in-time restore for SQL.

### Performance Efficiency
- Cache active case dashboards using `Azure Cache for Redis`.
- Use query tuning and indexing in Azure SQL to support case search and list views.
- Offload notifications to asynchronous Service Bus processing.
- Autoscale frontend and API tiers by CPU, request queue, and latency.

### Cost Optimisation
- Favor `Azure App Service` Standard/Premium v3 or `Container Apps` rather than dedicated VMs.
- Use `Azure SQL Database` Serverless or Hyperscale only if workload spikes require it; otherwise use General Purpose with reserved compute sizing.
- Apply autoscale to scale down outside peak business hours.
- Use `Azure Communication Services` pay-as-you-go for notification transport and avoid custom telecom plumbing.

### Operational Excellence
- Implement deployment pipelines with GitHub Actions / Azure DevOps.
- Use Application Insights alerts for failed case updates, slow response times, and Service Bus dead-letter growth.
- Document runbooks for common failures: SQL concurrency, failed notifications, identity issues.
- Use tags for resource owner, environment, and compliance scope.

---

## 5. Security & Governance (Policy-Aligned)

**RBAC and Identity**
- Define Azure AD groups: `Agents`, `Supervisors`, `CaseParticipants`, `Auditors`
- Assign least-privilege roles: e.g. `App Service Contributor` only for platform ops, `SQL DB Contributor` for database ops if needed.
- Use Managed Identity for apps to access SQL, Service Bus, and Key Vault.

**Network Security**
- Use `Virtual Network` with subnet isolation for App Service Environment or integrate Container Apps with VNet.
- Apply `Network Security Groups` for inbound/outbound flow control.
- Use private endpoints for Azure SQL, Service Bus, and Key Vault.

**Azure Policy Recommendations**
- Enforce allowed locations to UK South/UK West.
- Require Tagging policy for `Environment`, `Owner`, `BusinessUnit`.
- Enforce SQL auditing and Threat Detection.
- Block public IP exposure for PaaS resources.
- Require Key Vault soft delete and purge protection.

**Compliance**
- Keep all case data in UK regions to satisfy data residency.
- Enable SQL auditing and retain logs for investigation.
- Encrypt data at rest and in transit.

---

## 6. Scalability & Performance Strategy

**Scaling Approach**
- Frontend/API: horizontal scaling via App Service autoscale rules and Container Apps revisions.
- Cache: `Azure Cache for Redis` scales independently for read-heavy workloads.
- Notifications: `Service Bus` with partitioned queues for throughput.
- Database: scale-out read replicas if reporting load grows.

**Performance Considerations**
- Use selective case load and lazy-loading for client UI.
- Search using filtered SQL indexes and optionally Azure Cognitive Search if query complexity increases.
- Offload heavy report generation to background functions.
- Monitor latency at the API and SignalR boundary.

---

## 7. Reliability & Availability

**High Availability Design**
- Primary deployment in `UK South`, secondary failover in `UK West`.
- Use zone-redundant App Service / Container Apps and SQL to survive AZ outages.
- Use Service Bus with Geo-Disaster Recovery if cross-region failover is required.

**Failover Strategy**
- Active-active or active-passive failover for App Service with traffic manager or Front Door if global continuity is needed.
- SQL geo-restore to secondary region for data recovery.
- Use health probes and automated alerting for failover readiness.

---

## 8. Cost Considerations (Pricing-Informed)

**Relative cost drivers**
- `Azure SQL Database` is the largest steady cost due to data volumes and HA.
- `Azure Communication Services` adds variable cost for SMS/email volume.
- App hosting costs depend on scale settings; autoscaling minimizes waste.
- `Azure Cache for Redis` and `Service Bus` add moderate operational costs.

**Optimisation opportunities**
- Use `Azure SQL Elastic Pool` if multiple similar databases are needed.
- Pick `App Service Plan` size matched to business hours, with autoscale down overnight.
- Configure `Azure Communication Services` templates and batching to reduce notification send volume.
- Use `Azure Monitor` log retention carefully; keep only required data to limit ingestion costs.

**Estimate**
- Application PaaS hosting: approx £300–£900/month for a single Standard/Premium App Service plan with moderate traffic.
- Azure SQL Database General Purpose zone redundant: approx £500–£1,200/month depending on DTUs/vCores.
- Azure Communication Services notifications: £0.10–£0.30 per SMS equivalent message, email costs around £0.001 per message.
- Azure Cache for Redis Basic/Standard: approx £100–£250/month.
- These estimates are relative and should be validated using Azure Pricing Calculator for exact UK-region rates.

---

## 9. Risks & Recommendations (Advisor-Informed)

**Key risks**
- Over-reliance on a single region without a tested failover path.
- Notification delivery failure due to unhandled transient errors or telecom provider issues.
- Excessive SQL cost from unbounded scale or long-running analytics queries.
- Security gaps if external stakeholder access is not separated from internal agent workflows.

**Recommended mitigations**
- Implement geo-failover planning and regular restore drills.
- Use Service Bus retry policies, dead-letter handling, and alerting for notification failures.
- Monitor SQL DTU/compute usage and right-size with performance tuning.
- Isolate external portal access in a separate authentication flow and enforce conditional access.

---

## 10. Software Bill of Materials (SBOM) / Key Technology Stack

- Azure App Service or Azure Container Apps
- Azure SQL Database
- Azure Communication Services
- Azure SignalR Service
- Azure Service Bus
- Azure Cache for Redis
- Azure Active Directory
- Azure Key Vault
- Azure Monitor / Application Insights
- Azure Policy
- GitHub Actions / Azure DevOps pipeline
- Optional: Azure API Management
- Optional: Azure Cognitive Search

---

## 11. Assumptions

- Case data and stakeholder updates must remain in UK data residency.
- Agents and stakeholders require authenticated, role-based access.
- Notifications are primarily email/SMS, with real-time web updates for internal users.
- Expected load is moderate; architecture can scale as adoption grows.
- No existing legacy on-prem case system is being modernized in this initial design.
- Regulatory or sector-specific compliance beyond UK residency is not yet defined.

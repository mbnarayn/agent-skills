# Azure High-Level Design (HLD): Case Management Platform

**Document Version:** 1.0  
**Date:** May 13, 2026  
**Architect:** Azure Solutions Architecture  
**Classification:** Internal Use  

---

## 1. Overview

The **Case Management Platform** is an internal-facing PaaS application designed to enable support agents and supervisors to efficiently manage victim-related cases. The system provides case creation, status tracking, search/filtering, audit history, and role-based access control—all built on a secure, scalable Azure foundation.

**Scope:** Application architecture, infrastructure, security, and operational considerations for a moderate-availability system serving 50-200 internal users managing 10K-50K cases annually over 7-10 year retention periods.

**Key Architectural Goals:**
- Minimize operational overhead via PaaS services
- Ensure secure, audit-compliant case data handling
- Support scalable user and data growth
- Align to enterprise governance and security standards
- Maintain moderate availability during business hours with resilience

---

## 2. Architecture Summary

### Design Approach

The architecture follows a **layered PaaS approach** optimizing for **security, compliance, and operational simplicity**:

- **Presentation:** Azure App Service hosting a modern SPA (React/Vue) with Node.js API backend
- **Identity & Access:** Azure AD (Entra ID) with MFA for authentication; application-layer RBAC for agents/supervisors
- **Data Storage:** Azure SQL Database (transactional cases, updates, audit logs) + Cosmos DB (full-text search index)
- **Document Storage:** Azure Blob Storage for case files and attachments with lifecycle policies
- **Secrets & Keys:** Azure Key Vault for centralized credential and encryption key management
- **Networking:** Virtual Network with private endpoints, NSGs restricting access to internal users only
- **Monitoring:** Application Insights + Azure Monitor for observability and alerting

### Architectural Patterns

1. **Layered Architecture** (presentation, API, business logic, data tiers)
2. **Database per Tier** (transactional DB for cases, NoSQL for search optimization)
3. **Private Network Isolation** (VNet, private endpoints, no public IPs)
4. **Identity-First Security** (Managed Identity, RBAC, Zero Trust)
5. **Immutable Audit Logs** (compliance-grade tamper-proof logging)

---

## 3. Service Selection & Rationale (Cloud Architect)

| Service | Use Case | Rationale |
|---------|----------|-----------|
| **Azure App Service** | Web application hosting | PaaS choice: native RBAC integration, auto-scaling, built-in monitoring. Avoids VM management overhead. |
| **Azure SQL Database** | Case CRUD operations, audit logs | Relational data (structured cases, updates, audit trails). Transparent Data Encryption, automated backups, Point-in-Time Restore for compliance. |
| **Azure Cosmos DB** | Full-text search index, metadata cache | NoSQL flexibility for denormalized search index; globally consistent reads; serverless autoscaling aligns with moderate traffic patterns. |
| **Azure Blob Storage** | Case documents, attachments | Managed file storage; supports lifecycle policies for archiving old documents; geo-redundant replicas. |
| **Azure Key Vault** | Secrets, encryption keys, certificates | Centralized credential store; enforces access policies; audit logging; purge protection for compliance. |
| **Azure AD / Entra ID** | Authentication, authorization, MFA | Enterprise identity provider; built-in MFA; conditional access; integration with App Service; single sign-on. |
| **Azure Virtual Network** | Network isolation, security boundary | Private endpoints for SQL/Vault/Blob; NSGs restrict inbound traffic to internal corporate network only. |
| **Azure API Management** | API gateway, rate limiting, versioning | Provides API contract versioning, rate-limiting, request/response logging. Decouples API evolution from client updates. |
| **Application Insights** | Application monitoring, diagnostics | Built-in telemetry collection; dependency tracking; custom event tracking for business metrics (case creation, search latency). |
| **Azure Monitor** | Infrastructure monitoring, alerting | Platform-level metrics (App Service CPU/memory, SQL DTU usage, Cosmos RU consumption). Alert integration with incident management. |

**Deferred Services (Not Required for MVP):**
- Azure Service Bus / Event Grid: Case event publishing (future for audit consumers)
- Azure Functions: Scheduled tasks (e.g., archive old cases monthly)
- Azure Log Analytics Workspace: Advanced security analytics (SIEM integration future state)

---

## 4. High-Level Architecture Description

### Logical Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  INTERNAL AGENTS / SUPERVISORS               │
│               (Corporate Network / VPN Access)              │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTPS / TLS 1.2+
                    ┌────────▼─────────┐
                    │  Azure AD / MFA  │
                    │   (Entra ID)     │
                    └────────┬─────────┘
                             │ Identity Token
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼──────────┐  ┌─────▼────────┐  ┌───────▼────────┐
   │   App Service │  │   API Mgmt   │  │  Portal/Docs   │
   │ (React SPA +  │  │ (Rate Limit, │  │  (Admin Panel) │
   │  Node.js API) │  │ Versioning)  │  │                │
   └────┬──────────┘  └────────┬─────┘  └────────┬───────┘
        │                      │                 │
        └──────────┬───────────┴────────┬────────┘
                   │ Managed Identity   │
        ┌──────────▼──────────────────┐
        │                             │
   ┌────▼──────────┐  ┌─────────────┐│
   │  Azure SQL    │  │ Application ││
   │  Database     │  │  Insights   ││
   │               │  │             ││
   │ • Cases       │  │ • Logs      ││
   │ • Updates     │  │ • Events    ││
   │ • Audit Logs  │  │ • Metrics   ││
   │ • Users/RBAC  │  │             ││
   └────┬──────────┘  └─────────────┘│
        │                            │
        │ Private Endpoint           │
        │ (NSG restricted)           │
   ┌────▼──────────────────────────┐ │
   │      Azure Virtual Network     │ │
   │   (Private Endpoints, NSGs)    │ │
   │                                │ │
   └────┬──────────────┬────────┬──┘ │
        │              │        │    │
   ┌────▼───┐  ┌──────▼──┐  ┌──▼──┐ │
   │SQL DB  │  │Cosmos DB│  │Vault│ │
   │Private │  │Private  │  │Private
   │Endpoint│  │Endpoint │  │Endpoint
   └────────┘  └─────────┘  └──────┘
        │              │        │
   ┌────▼───────────────┴────────┴─────┐
   │   Azure Blob Storage (Docs)       │
   │  (Private Endpoint, Lifecycle)    │
   └───────────────────────────────────┘
```

### Component Interactions

1. **Agent Login:** Agent accesses App Service → Azure AD MFA → Token issued
2. **Case Operations:** 
   - Create/update case → API → SQL DB (transactional write) + Cosmos DB index (search metadata)
   - Audit log entry auto-created (timestamp, user, changes)
3. **Search:** Agent queries → API → Cosmos DB full-text search → SQL DB detail fetch
4. **Document Upload:** Agent uploads file → API → Blob Storage (private endpoint) → metadata in SQL
5. **Monitoring:** App Service/SQL/Cosmos metrics → Application Insights → Azure Monitor alerts

---

## 5. Well-Architected Assessment

### Security
✅ **MFA & Authentication:** Azure AD enforces MFA (TOTP or Authenticator app); no password-only access  
✅ **Managed Identity:** App Service → Key Vault/SQL/Blob via managed identity (no secrets in code/config)  
✅ **RBAC (Dual Layer):**
   - Management plane: Azure Resource roles (least privilege for infra teams)
   - Application plane: Custom roles (Agent, Supervisor, Admin) with permission matrix
✅ **Encryption:**
   - In-transit: HTTPS/TLS 1.2+ enforced by API Management
   - At-rest: SQL TDE with customer-managed key in Key Vault; Blob storage-side encryption
✅ **Network Isolation:** VNet with NSGs; no public endpoints; private endpoints for all data services  
✅ **Secrets Management:** All credentials (DB connection strings, API keys) stored in Key Vault with audit logging  
⚠️ **Trade-off:** No advanced threat detection (Azure Defender for SQL) in MVP to minimize costs; upgrade if breach risk profile increases

### Reliability
✅ **Availability:** App Service configured with 2+ instances (auto-scale min: 2, max: 5) across multiple zones  
✅ **Database Resilience:** SQL Database geo-replication enabled for failover; automatic backups (7-day default, extendable to 35 days)  
✅ **Fault Tolerance:** Cosmos DB multi-region replicas for search index; can tolerate regional outages  
✅ **Health Monitoring:** Application Insights tracks dependency health (SQL, Cosmos availability)  
✅ **Disaster Recovery:** Quarterly backup restore tests scheduled  
⚠️ **Trade-off:** Moderate availability (business-hours critical acceptable); full 99.99% SLA deferred to Phase 2 if justified by business metrics

### Performance Efficiency
✅ **Auto-scaling:** App Service scales horizontally based on CPU/memory thresholds  
✅ **Database Optimization:** SQL Database query indexing on case status, agent ID, date ranges; statistics auto-update  
✅ **Caching Strategy:** Cosmos DB search metadata denormalized; Redis cache optional for high-concurrency scenarios (>50 concurrent users)  
✅ **Async Processing:** Long-running operations (document OCR, bulk exports) can defer to Azure Functions (Phase 2)  
✅ **Monitoring Thresholds:** Application Insights tracks API latency (target <500ms, p95); alerts if >750ms  
⚠️ **Trade-off:** Full-text search (Cosmos) is eventually consistent; real-time search accuracy deferred slightly for scalability

### Cost Optimization
✅ **PaaS Selection:** Avoids IaaS overhead (no VMs to patch/manage); service tiers right-sized for moderate load  
✅ **Reserved Instances:** Recommend 1-year RI on App Service (20% savings) and SQL Database (25% savings)  
✅ **Data Lifecycle:** Blob Storage hot tier (active documents) → cool tier (90+ days) → archive tier (7+ years)  
✅ **Monitoring Budget:** Basic Application Insights tier; advanced analytics deferred  
✅ **Database Sizing:** Start with SQL Standard S1 (~$25/day); scale to S2/S3 based on actual DTU consumption  
⚠️ **Trade-off:** Cosmos DB for search adds cost (~$50-100/month); alternative is SQL FTS (full-text search) in-database, lower cost but less scalable

### Operational Excellence
✅ **Infrastructure as Code:** All resources defined in Bicep (version-controlled, reproducible deployments)  
✅ **CI/CD Pipeline:** GitHub Actions / Azure DevOps automated deployments (dev → staging → production)  
✅ **Monitoring & Alerting:** Application Insights + Azure Monitor send alerts to on-call via Teams/PagerDuty  
✅ **Documentation:** Runbooks for manual failover, key rotation, incident response  
✅ **Audit Trail:** All infrastructure changes logged via Azure Policy audit events  
✅ **Backup & Recovery:** SQL automated backups; quarterly restore tests; RTO/RPO defined  

---

## 6. Security & Governance (Policy-Aligned)

### Identity & Access Control

**Authentication Layer:**
- Azure AD / Entra ID enforces MFA (Microsoft Authenticator or TOTP)
- Conditional Access policies can enforce device compliance, location restrictions
- Session timeout: 8 hours (configurable per org policy)

**Authorization Layer:**
```
Application Roles:
├─ Agent (default)
│  ├─ Create cases (own)
│  ├─ Update cases (own)
│  ├─ View assigned cases
│  ├─ Search cases (metadata only, not PII export)
│  └─ Cannot: Delete, reassign, view other agents' cases
├─ Supervisor
│  ├─ Create/update/delete any case
│  ├─ Reassign cases to agents
│  ├─ Generate reports (case metrics, agent performance)
│  ├─ View audit logs (filtered)
│  └─ Cannot: System configuration, user management
└─ Admin
   ├─ All permissions
   ├─ User onboarding/offboarding
   ├─ System configuration (retention policies, integrations)
   └─ Full audit log access
```

**Managed Identity:**
- App Service → SQL Database: Managed Identity (no connection string exposed)
- App Service → Key Vault: Managed Identity for secret retrieval
- App Service → Blob Storage: Managed Identity for document access

### Data Protection

**Encryption in Transit:**
- API Management enforces HTTPS/TLS 1.2+ (TLS 1.3 recommended)
- Service-to-service communication uses managed identities (encrypted by default)
- VPN required for on-premises agents (corporate network gateway)

**Encryption at Rest:**
- SQL Database: Transparent Data Encryption (TDE) with customer-managed key in Key Vault
- Cosmos DB: Client-side encryption for sensitive fields (victim names, contact info, case notes)
- Blob Storage: Server-side encryption with customer-managed keys
- Key Vault: Premium tier (optional for FIPS 140-2 compliance if required by org policy)

**Key Management:**
- Encryption keys rotated every 90 days (automated in Key Vault)
- Key access logged in Key Vault audit events
- Purge protection enabled; soft-delete retention 90 days

### Network Security

**Virtual Network Design:**
- Private subnet for App Service (internal only)
- Private subnets for SQL Database, Cosmos DB, Blob Storage (private endpoints)
- NSGs restrict inbound to corporate network CIDR ranges only
- Outbound restricted to approved destinations (no internet access from data layer)

**Access Control:**
- No public endpoints exposed; all services accessed via private endpoints
- Azure Firewall (optional) for DDoS protection and centralized logging

### Audit & Compliance

**SQL Audit:**
- Enable SQL Database audit on all tables
- Log SELECT, INSERT, UPDATE, DELETE operations
- Audit logs sent to Azure Storage (immutable archive)
- Retention: 7-10 years for victim protection and litigation holds

**Application Audit:**
- Case creation/update events logged: timestamp, user ID, case ID, old/new values
- Access logs: who viewed sensitive fields (victim PII)
- Failed authentication attempts logged
- Compliance reports generated quarterly (user access reviews, data deletion confirmations)

**Azure Policy:**
- Enforce TLS 1.2+ on all resources
- Require private endpoints for SQL/Storage/Vault
- Deny public IP assignments
- Audit missing MFA policies
- Enforce resource tagging (cost center, environment, owner)

---

## 7. Scalability & Performance Strategy

### Horizontal Scaling

**App Service:**
- Auto-scaling rules: CPU >70% → scale out; CPU <30% → scale in
- Min instances: 2 (HA baseline)
- Max instances: 5 (covers peak 50 concurrent users with headroom)
- Scale-out time: ~3-5 minutes (acceptable for moderate load)

**Database:**
- SQL Database: Dynamic scaling (Standard S1 → S2 → S3) based on DTU consumption
- Cosmos DB: Autoscale provisioned throughput (400-2000 RU/s) for search index
- Read replicas: Optional failover replica in secondary region (geo-replication)

**Caching:**
- Application-level: In-memory cache (node.js built-in) for agent role definitions and search filters (~10 MB)
- Distributed cache (optional): Azure Cache for Redis if >50 concurrent users detected
- Query result caching: Case summary cache invalidated hourly

### Performance Optimization

**Database Query Performance:**
- Index creation on: case_status, agent_id, created_date, case_number
- Statistics auto-updated weekly
- Query plan analysis for long-running searches (Application Insights query telemetry)

**Search Optimization:**
- Cosmos DB denormalized index contains: case ID, status, summary, last updated, agent name
- Full-text search (case notes, victim name) uses SQL FTS with indexed views
- Search results paginated (25 cases per page) to avoid large result sets

**API Performance Targets:**
- Case creation: <200ms (p50), <500ms (p95)
- Case search: <300ms (p50), <750ms (p95)
- Case update: <150ms (p50), <400ms (p95)
- Monitored via Application Insights custom metrics

---

## 8. Reliability & Availability

### High-Availability Design

**Application Tier:**
- Multiple App Service instances (min 2) across availability zones
- Health probes monitor application readiness (heartbeat endpoint)
- Auto-failover triggered if instance fails (unhealthy)

**Database Tier:**
- SQL Database zone-redundant backup (GRS)
- Automatic failover to secondary replica (if geo-replicated)
- Point-in-Time Restore available for 35 days (configurable)

**Data Tier:**
- Cosmos DB configured for multi-region failover (3 replicas across regions)
- Blob Storage geo-redundant storage (GRS) for document replicas

### Failover & Recovery Strategy

**Application Failover:**
- App Service automatic failover (transparent to users)
- Session state not retained (requires re-login if instance fails)

**Database Failover:**
- SQL failover replica (secondary region) auto-promoted if primary fails (RTO ~30 seconds)
- Cosmos failover via automatic region switching

**Disaster Recovery:**
- RPO (Recovery Point Objective): 24 hours (daily SQL backups)
- RTO (Recovery Time Objective): 4 hours (restore from backup + reconfigure endpoints)
- Quarterly restore drills validate RTO/RPO assumptions

### Monitoring & Alerting

**Key Metrics:**
- App Service: CPU >80%, Memory >85%, Request count anomalies
- SQL Database: DTU usage >80%, Long-running queries (>5 seconds)
- Cosmos DB: RU consumption >80%, Replication lag >1 second
- Application: Error rate >1%, API latency p95 >750ms

**Alert Actions:**
- Critical: Page on-call engineer (PagerDuty integration)
- Warning: Email ops team
- Info: Log to Azure Monitor for dashboard view

---

## 9. Cost Considerations (Pricing-Informed)

### Estimated Monthly Cost Breakdown (50-200 agents, 10K-50K cases/year)

| Component | SKU | Estimated Monthly Cost |
|-----------|-----|------------------------|
| App Service | Standard B2 (2 cores, 3.5 GB RAM) | $80-150 |
| SQL Database | Standard S1 (20 DTU) | $200-300 |
| Cosmos DB | Autoscale 400-2000 RU/s | $80-150 |
| Blob Storage | Standard GRS, 1-10 GB active | $20-40 |
| Application Insights | Standard pay-as-you-go | $50-100 |
| Key Vault | Standard tier | $0.34 |
| API Management | Developer tier (10,000 calls/month free) | $50-100 |
| Azure Monitor | Logs retention (30 days) | $30-50 |
| Virtual Network | Standard | $20-30 |
| **Total (MVP)** | | **$600-950/month** |

### Cost Optimization Strategies

1. **Reserved Instances:** Purchase 1-year RIs on App Service (20-30% savings) + SQL Database (25-30% savings)
   - Estimated savings: ~$150-200/month → **Net: $400-750/month**

2. **Auto-scale Wisely:** Keep max instances at 5; scale down aggressively off-hours
   - Monitor actual peak load; right-size minimum instances

3. **Database Right-sizing:** Start with S1, monitor DTU usage; only scale to S2/S3 if >80% consumed

4. **Blob Lifecycle Policies:**
   - Active documents (0-90 days): Hot tier (~$0.0184/GB)
   - Warm archive (90-365 days): Cool tier (~$0.01/GB, 95% cheaper)
   - Cold archive (>365 days): Archive tier (~$0.004/GB, 99% cheaper)
   - Estimated storage savings: 70-80% via tiering

5. **Search Index Trade-off:** If budget critical, migrate from Cosmos DB to SQL Server full-text search (FTS)
   - Saves ~$100-150/month but degrades search performance at scale
   - Viable only if concurrent users <20

### Phase 2 Cost Drivers (If Added)
- Event Grid: Case event distribution (~$0.50 per million events)
- Azure Functions: Scheduled tasks/automation (~$0.001 per execution + storage)
- Advanced Threat Protection: Azure Defender for SQL (~$50/month per server)

---

## 10. Risks & Mitigations (Advisor-Informed)

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| **SQL Database hotspots under peak load** | Medium | High | Pre-create indexes on high-query columns; use query store for telemetry; consider partitioning audit logs by date |
| **Cosmos DB RU overrun** | Medium | Medium | Autoscale RU provisioning; implement request rate limiting in API; monitor RU/second trends |
| **Blob storage explosion (uncontrolled growth)** | Medium | Medium | Enforce lifecycle policies (archive after 1 year); document quota per agent (no unlimited uploads); alert if >100 GB |
| **Managed Identity misconfiguration** | Low | High | Regular access reviews (quarterly); audit Key Vault access logs; test failover scenarios |
| **Data residency violation** | Low | Critical | Enforce Azure Policy to reject resources outside approved regions; compliance audit annually |
| **Audit log immutability broken** | Low | Critical | Enable Blob Storage WORM (Write Once Read Many); regular integrity checks; no delete permissions on audit storage account |
| **Network segmentation bypass (insider threat)** | Low | High | NSG audit logs reviewed quarterly; conditional access enforces corporate network geofencing; consider Azure Sentinel for SIEM |
| **Key rotation failure** | Low | High | Automate key rotation in Key Vault; test rotation annually; alert on failed rotations |
| **Seasonal spike (case surge)** | Medium | Medium | Validate auto-scale rules; load test 2x peak capacity; maintain runbook for manual scale if auto-scale lags |
| **Compliance audit findings** | Low | Medium | Conduct annual compliance assessments; track Azure Policy audit results; maintain audit trail documentation |

### Recommended Mitigations (Priority Order)

1. **Implement Azure Policy** (Week 1): Enforce TLS, private endpoints, MFA, tagging
2. **Configure SQL Audit + Blob Archive** (Week 1-2): Enable immutable audit trail
3. **Load Test** (Week 3): Validate auto-scale at 2x peak load
4. **Quarterly Compliance Review** (Ongoing): Access reviews, audit log analysis, policy violations

---

## 11. Software Bill of Materials (SBOM) / Key Technology Stack

### Azure Services (IaC/Deployment)
- **Compute:** Azure App Service (Node.js 20 LTS runtime)
- **Database:** Azure SQL Database (v12, latest API version)
- **NoSQL:** Azure Cosmos DB (API for MongoDB or SQL API)
- **Storage:** Azure Blob Storage (Standard GRS)
- **Identity:** Azure AD / Entra ID (Premium P1 recommended)
- **Secrets:** Azure Key Vault (Standard tier)
- **Networking:** Azure Virtual Network, NSGs, Private Endpoints
- **API Gateway:** Azure API Management (Developer tier, upgrade to Standard if >10K calls/day)
- **Monitoring:** Application Insights, Azure Monitor
- **IaC:** Bicep templates (version-controlled in Git)

### Application Stack
- **Frontend:** React 18+ (or Vue 3+) SPA
- **Backend:** Node.js 20 LTS, Express.js or Fastify
- **ORM:** TypeORM or Prisma (for SQL abstraction + audit triggers)
- **Search:** Cosmos DB SDK (Node.js) or SQL Server FTS
- **Auth:** Microsoft Authentication Library (MSAL) for Azure AD integration
- **Logging:** Winston or Pino (structured logging to Application Insights)
- **Testing:** Jest (unit), Cypress (E2E)

### CI/CD & DevOps
- **Source Control:** Git (GitHub or Azure DevOps)
- **CI/CD:** GitHub Actions or Azure Pipelines
- **Deployment:** Bicep templates via `az deployment group create`
- **Environment Management:** Dev, Staging, Production with separate resource groups

### Security Tools
- **Secrets Management:** Azure Key Vault CLI/SDK
- **Network Monitoring:** Azure Network Watcher
- **Compliance:** Azure Policy, Azure Blueprint (optional)
- **Threat Detection:** Azure Security Center (basic) or Azure Defender (advanced, future)

---

## 12. Assumptions

### Business Assumptions
1. ✅ **Case Volume:** 10K-50K cases annually (average 50-100 active cases per month)
2. ✅ **Concurrent Users:** Peak 20-50 concurrent agents/supervisors during business hours
3. ✅ **Business Hours:** System availability critical 8 AM–6 PM (weekdays); can tolerate outages outside hours
4. ✅ **Data Retention:** 7-10 years for litigation/compliance holds (victim protection)
5. ✅ **Audit Compliance:** Cases and all updates must be immutable and audited (no deletion, only archival)
6. ✅ **Geographic Scope:** Single region (US-East preferred, expandable to multi-region Phase 2)
7. ✅ **User Onboarding:** Manual (no self-service registration); managed by admins

### Technical Assumptions
1. ✅ **Network Connectivity:** Agents access platform via corporate VPN or on-premises network (no internet-facing)
2. ✅ **Authentication:** Azure AD / Entra ID is organization's identity provider
3. ✅ **Encryption Keys:** Organization IT manages Azure Key Vault access and key rotation policies
4. ✅ **Database Performance:** DTU consumption <80% under peak load (S1 sufficient for MVP)
5. ✅ **Document Size:** Average case file <10 MB; bulk uploads handled asynchronously (Phase 2)
6. ✅ **Search Latency:** <750ms acceptable (eventual consistency in Cosmos DB acceptable)
7. ✅ **Backup Frequency:** Daily SQL backups; geo-replication enabled for HA
8. ✅ **Monitoring:** Basic Application Insights tier sufficient; advanced SIEM integration future

### Organizational Assumptions
1. ✅ **Azure Subscription:** Dedicated subscription or shared with clear resource group isolation
2. ✅ **Governance:** Organization has Azure Policy framework; approved regions/SKUs defined
3. ✅ **Cost Center:** Budget allocated for infrastructure (~$600-950/month MVP + reserve for growth)
4. ✅ **Change Management:** Organization has CAB (Change Advisory Board) for production deployments
5. ✅ **Incident Response:** Defined SLA/RTO/RPO; on-call rotation established for critical alerts
6. ✅ **Compliance:** HIPAA/SOC 2 compliance framework understood; audit trails non-negotiable

---

## Next Steps & Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- [ ] Approval of HLD by architecture review board
- [ ] Create Azure Resource Group and Key Vault
- [ ] Define Azure Policy enforcement (TLS, private endpoints, tagging)
- [ ] Set up SQL Database with schema (Cases, CaseUpdates, AuditLog, Users, Roles tables)
- [ ] Configure Application Insights and monitoring dashboards

### Phase 2: Identity & Security (Weeks 3-4)
- [ ] Azure AD app registration and RBAC configuration
- [ ] Configure MFA enforcement and conditional access policies
- [ ] Set up Virtual Network and private endpoints
- [ ] Deploy API Management with rate limiting and versioning policies

### Phase 3: Application & Data (Weeks 5-6)
- [ ] Deploy App Service and configure auto-scaling
- [ ] Publish web application (React SPA + Node.js API)
- [ ] Integrate Azure AD authentication with MSAL
- [ ] Deploy Cosmos DB search index and configure full-text search

### Phase 4: Compliance & Operations (Weeks 7-8)
- [ ] Enable SQL Audit and configure blob archive for audit logs
- [ ] Deploy monitoring alerts and test incident response runbooks
- [ ] Conduct load testing (2x peak capacity validation)
- [ ] Perform security assessment and penetration testing

### Phase 5: Pilot & Go-Live (Weeks 9-10)
- [ ] Execute pilot launch with supervisor group (closed beta)
- [ ] Collect feedback and fix issues
- [ ] Plan full rollout communication and training
- [ ] Monitor closely during first 2 weeks post-launch

---

## Document Approval & Sign-Off

**Architecture Review:** [  ] Approved  |  [  ] Changes Requested  |  [  ] Rejected

**Approver:** ________________  
**Date:** ________________  

**Security Review:** [  ] Approved  |  [  ] Changes Requested  |  [  ] Rejected

**Approver:** ________________  
**Date:** ________________  

---

**End of HLD Document**

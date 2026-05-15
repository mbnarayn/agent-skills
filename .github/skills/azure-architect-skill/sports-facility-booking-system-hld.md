# Sports Facility Booking System - Azure HLD

## 1. Overview

- Purpose: Deliver a scalable, secure Sports Facility Booking System for users to browse venues, book time slots, reschedule, and cancel bookings.
- Scope: Mobile/web app front-end, booking workflow, facility inventory, availability engine, payment/notification integration, and administrator operations.
- System summary: PaaS-first Azure solution using Azure UK regions for data residency, with real-time availability, event-driven booking updates, and strong governance.

## 2. Architecture Summary

- High-level design:
  - Mobile/web client → API layer → booking domain services → data/store + event processing.
  - Use API Management + Azure App Service for APIs, Azure Functions for booking workflow, Cosmos DB for session/booking data, and Event Grid for notifications.
- Architectural patterns:
  - Microservices-ish domain separation (Bookings, Facilities, Users, Payments)
  - Event-driven orchestration for reschedule/cancel flows
  - Command query separation: write operations via APIs/functions, reads via Cosmos DB materialized views
  - Managed identity and zero-trust networking

## 3. Service Selection & Rationale (Cloud Architect)

- Azure API Management + App Service Environment:
  - Exposes booking API and protects endpoints.
  - Rationale: central policy enforcement, rate limiting, OAuth token validation.
- Azure App Service (Linux) or Azure Container Apps for API backend:
  - Hosts REST API for booking/reschedule/cancel and facility browsing.
  - Rationale: PaaS reduces ops overhead, supports autoscale and deployment slots.
- Azure Functions:
  - Handles workflow events: availability check, hold, confirmation, reschedule, cancellation, reminder notifications.
  - Rationale: event-driven compute for bursty booking activity with cost efficiency.
- Azure Cosmos DB (Core SQL API) in UK South / UK West:
  - Stores bookings, users, facilities, availability snapshots.
  - Rationale: low latency, global distribution optional, schema flexibility for evolving booking metadata.
- Azure Cache for Redis:
  - Caches availability windows, facility metadata, and rate-limit state.
  - Rationale: improves read performance and reduces pressure on Cosmos DB during peak booking windows.
- Event Grid + Service Bus:
  - Event Grid for domain events (BookingCreated, BookingUpdated, BookingCancelled)
  - Service Bus for durable workflow and payment integration.
  - Rationale: reliable asynchronous processing, decoupling between booking engine and notifications/payments.
- Azure Cognitive Search (optional):
  - Enables fast facility search and filtering by location, availability, equipment.
  - Rationale: better user experience for browsing facilities.
- Azure Notification Hubs or Logic Apps:
  - Sends push notifications / email reminders on booking changes.
  - Rationale: managed messaging service simplifies cross-platform notifications.
- Azure Key Vault:
  - Stores secrets, API keys, certificates.
  - Rationale: no application secrets in code.
- Azure Monitor + Application Insights:
  - Observability for API performance, function execution, failure rates.
  - Rationale: operational excellence and proactive issue detection.

## 4. Well-Architected Assessment

- Security:
  - Identity-first access using Azure AD B2C for end users.
  - RBAC for admin/ops in Azure Portal.
  - Key Vault for secrets; App Service / Functions use managed identities.
- Reliability:
  - Booking workflows use Service Bus dead-lettering and retries.
  - Cosmos DB multi-region failover in UK South / UK West if required.
- Performance Efficiency:
  - Redis caching for hot facility availability.
  - Autoscale App Service / Function plans based on queue depth and CPU.
- Cost Optimisation:
  - Serverless compute for unpredictable booking peaks.
  - Use App Service Premiumv3 or Container Apps reserved plan only if traffic justifies it.
  - Cosmos DB autoscale and provisioned throughput tuned to query patterns.
- Operational Excellence:
  - Deployment via Azure DevOps/GitHub Actions with ARM/Bicep templates.
  - Health probes, alerts, and runbooks for failed booking flows.

## 5. Security & Governance (Policy-Aligned)

- RBAC:
  - `Contributor` restricted to service owners.
  - `Reader` for reporting teams.
  - Custom roles for support staff to manage bookings without infrastructure access.
- Managed Identity:
  - App Service and Functions use system-assigned identities to access Cosmos DB, Key Vault, Service Bus.
- Network security:
  - Private endpoints for Cosmos DB and Key Vault.
  - App Service outbound traffic restricted via VNet integration and Service Endpoints.
- Azure Policy recommendations:
  - Enforce immutable storage account versioning and secure transfer.
  - Require Key Vault integration for App Service and Functions.
  - Enforce TLS 1.2+ on App Service.
  - Audit public network access on data stores.
- Governance:
  - Tag resources with `Environment`, `Application`, `CostCenter`, `DataClassification`.
  - Use Azure Policy for tagging and resource location (UK regions only).

## 6. Scalability & Performance Strategy

- Scale approach:
  - Horizontal scaling for App Service / Container Apps and Functions.
  - Redis + Cosmos DB read replicas or multi-region reads for heavy browsing traffic.
- Performance considerations:
  - Minimize read latency with Redis caching of facility availability and booking status.
  - Use Cosmos DB partitioning on `facilityId` or `bookingDate` for balanced throughput.
  - Avoid hot partitions by sharding booking documents across facility-date keys.
- Load pattern handling:
  - Peak bookings around scheduling windows handled by autoscaling and queue-backed functions.
  - Burst capacity through Azure Functions Consumption or Premium plans.

## 7. Reliability & Availability

- High availability:
  - Deploy primary workloads in UK South, with optional secondary UK West for business continuity.
  - Use availability zones for Cosmos DB and App Service if required.
- Failover strategy:
  - Cosmos DB active geo-replication configured for UK West fallback.
  - API endpoint failover via Traffic Manager or Azure Front Door if multi-region active/passive is needed.
- Data durability:
  - Use Cosmos DB point-in-time restore and backup retention.
  - Persist booking state in durable event streams (Service Bus/Event Grid) for replay on recovery.
- Fault isolation:
  - Separate booking, notification, and payment processing into distinct services to prevent cascading failures.

## 8. Cost Considerations (Pricing-Informed)

- Relative cost drivers:
  - Cosmos DB RU/s is the largest cost for frequent booking queries and writes.
  - App Service / Functions compute costs are moderate if autoscaled correctly.
  - Event Grid / Service Bus costs are low but rise with very high event volume.
- Optimization opportunities:
  - Use Cosmos DB autoscale with a cap matched to peak booking windows.
  - Cache more aggressively in Redis to reduce RU consumption.
  - Use Azure Functions Consumption tier for non-constant workflow workloads.
  - Consider App Service Standard for moderate predictable traffic; move to Premium only if required by VNet integration and scale.
- Estimate:
  - Small production footprint: ~£800–£1,200/month with Cosmos DB single-region, App Service, Redis, and monitoring.
  - Medium production with reserve capacity and geo-failover: ~£1,500–£2,500/month.
  - These are relative ranges; exact pricing requires UK region calculator data for RU/s and instance sizing.

## 9. Risks & Recommendations (Advisor-Informed)

- Risk: Booking conflicts during simultaneous updates.
  - Mitigation: enforce optimistic concurrency / distributed lock on facility slot, use Service Bus for sequential booking finalization.
- Risk: Hot partition or RU spikes in Cosmos DB.
  - Mitigation: partition by facility-date and use Redis to absorb read traffic.
- Risk: User session data leak or secret exposure.
  - Mitigation: Key Vault + managed identities, no secrets in code/config.
- Risk: Operational drift and ungoverned resources.
  - Mitigation: enforce Azure Policy, infrastructure as code, and resource tagging.
- Risk: Notification delivery failures causing missed booking reminders.
  - Mitigation: retry policy with dead-lettering and monitoring for notification failure rates.

## 10. Software Bill of Materials (SBOM) / Key Technology Stack

- Front-end: React Native / Flutter mobile app, React SPA for web
- API: .NET 8 / Node.js REST API hosted in Azure App Service or Container Apps
- Workflow: Azure Functions (.NET or JavaScript)
- Database: Azure Cosmos DB (Core SQL API)
- Cache: Azure Cache for Redis
- Messaging: Azure Event Grid + Azure Service Bus
- Identity: Azure AD B2C for customers; Azure AD for administrators
- Secrets: Azure Key Vault
- Observability: Azure Monitor, Application Insights, Log Analytics
- Deployment: Azure DevOps / GitHub Actions + Bicep

## 11. Assumptions

- Data residency requirement: UK-only.
- Users require bookings, rescheduling, cancellations, and reminders.
- Payments are integrated but may be handled by a third-party gateway.
- Facility inventory is moderate in size and changes regularly.
- The system must support peak booking periods with automatic scaling and low-latency availability lookups.

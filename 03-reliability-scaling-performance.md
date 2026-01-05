# Plugilo Infrastructure

This document outlines the reliability, scaling, and performance architecture design for Plugilo's global SaaS platform.

## Part 3 – Reliability, Scaling & Performance

### 1. Context and Requirements

**Platform:**
- 10–15 million users
- Microservices architecture
- Backend services in Node.js / Python / Java
- Databases: PostgreSQL, MongoDB, Elasticsearch, Redis
- AI workloads (GPU optional; no deep ML required)
- Web & mobile clients (Web, iOS, Android)
- Deployed on Azure (primary) with multi-cloud connectivity to AWS and GCP

**Key Requirements:**
- Handle 10× traffic spikes
- Zero or near-zero downtime deployments
- Quick recovery from failures
- High availability across regions
- Performance optimization

---

## 2. Traffic Spike Handling (10× Traffic)

### 2.1 Multi-Layer Scaling Strategy

**Multi-Layer Scaling Strategy:**
1. **CDN Layer**: Azure Front Door automatically scales (no action required)
2. **Application Layer**: KEDA/HPA scales pods within 2-3 minutes
3. **Node Layer**: Karpenter provisions nodes within 30 seconds
4. **Database Layer**: Read replicas and connection pooling (PgBouncer)
5. **Burst Capacity**: Azure Container Instances (ACI) for immediate capacity during extreme spikes

**Bottleneck Identification:**
- **First Bottleneck**: Database connection pool (mitigated with read replicas and PgBouncer)
- **Second Bottleneck**: Application pod startup time (mitigated with pre-warmed pods, health checks, ACI)
- **Third Bottleneck**: Node provisioning (mitigated with Karpenter fast provisioning and ACI)

### 2.2 Performance Testing

**Decision: Comprehensive Performance Testing Before Traffic Spikes**

**Tool: K6** (integrated in CI/CD pipeline)
- **Test Structure**: Modular test scripts with reusable scenarios (similar to page object model for API/load testing)
- **Test Types**: Load (10× traffic), stress (15×, 20×), spike (0 to 10× in minutes), endurance (sustained high load)
- **Scenarios**: Baseline capacity, 10× traffic simulation, failure scenarios under load, recovery testing

**Ramp-Up Strategy for 10-15M Users (10% CCU = 1-1.5M concurrent users):**
- **Baseline Test**: 100K VUs (Virtual Users) over 5 minutes ramp-up, sustain 10 minutes
- **10× Traffic Test**: 1M VUs over 15 minutes ramp-up (linear: 0 → 1M), sustain 30 minutes, ramp-down 10 minutes
- **Spike Test**: 1.5M VUs over 2 minutes ramp-up (aggressive spike), sustain 5 minutes
- **Endurance Test**: 1M VUs over 20 minutes ramp-up, sustain 2 hours
- **Ramp-Up Pattern**: Linear ramp-up for load testing, stepped ramp-up (10% increments every 2 minutes) for stress testing

**Metrics**: P50/P95/P99 latency, RPS throughput, error rate, resource utilization (CPU/memory/network), scaling behavior (time to scale, effectiveness)
- **Frequency**: Before major releases, quarterly capacity validation, after infrastructure changes, continuous in CI/CD

### 2.3 Overprovisioning Strategy

**Decision: Kubernetes Node Overprovisioning with Spot Instances for Cost Optimization**

**Implementation**: [Kubernetes Node Overprovisioning](https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/) using placeholder pods with negative PriorityClass
- **Placeholder Pods**: Low-priority pods (PriorityClass: -1000) that reserve capacity and get preempted when real workloads need resources
- **Spot Instances**: Placeholder pods run on spot nodes (70% cost savings) via node selector/tolerations
- **Resource Reservation**: 20-30% buffer capacity (e.g., 5 placeholder pods × 0.1 CPU, 200Mi memory = 0.5 CPU, 1GiB total)
- **Pod Anti-Affinity**: Distribute placeholder pods across nodes to reserve capacity on multiple nodes

**Benefits:**
- **Immediate Scheduling**: New pods schedule instantly (no pending state during scale-up)
- **Cost Optimization**: Spot instances reduce overprovisioning cost by 70% vs on-demand
- **Automatic Preemption**: Placeholder pods automatically preempted when real workloads need capacity
- **Karpenter Integration**: Karpenter treats preferred affinity as hard rules, ensuring minimum node count

**Configuration:**
- **PriorityClass**: Negative priority (-1000) for placeholder pods
- **Node Selector**: Placeholder pods target spot instance node pools
- **Replica Count**: Scale based on desired buffer capacity (e.g., 5 replicas for 0.5 CPU, 1GiB buffer)
- **Resource Requests**: Configure per-pod requests to match desired total buffer (replicas × per-pod requests = total buffer)

**Use Cases**: Known peak periods (Black Friday, product launches), critical services requiring instant scaling, seasonal traffic patterns

### 2.4 Azure Container Instances (ACI) for Burst Capacity

**Decision: Azure Container Instances for Immediate Burst Capacity**

**Implementation**: [Burst to Azure Container Instances](https://learn.microsoft.com/azure/aks/concepts-scale#burst-to-azure-container-instances-aci) via Virtual Nodes (Virtual Kubelet)
- **Virtual Nodes**: ACI appears as virtual Kubernetes nodes in AKS cluster, enabling transparent pod scheduling to ACI
- **Scaling Flow**: KEDA/HPA scales pods → AKS nodes scale first → If AKS at capacity, pods schedule to ACI virtual nodes → Traffic routes automatically
- **Container Images**: Same images as AKS (no application changes required)
- **Networking**: Virtual nodes deployed to separate subnet in same VNet, secured traffic between ACI and AKS

**Benefits**: Instant capacity (seconds vs 2-3 minutes for node provisioning), pay-per-second billing (no idle node costs), seamless integration (transparent to applications)
- **Use Cases**: Unexpected traffic spikes, Black Friday events, geographic burst capacity
- **Trade-offs**: Higher cost for sustained traffic (per-second billing), stateless only, requires Virtual Kubelet setup

### 2.5 Proactive Scaling

**Decision: Predictive Scaling Based on Historical Patterns and Timestamps**

**Implementation**: KEDA CronScaler for scheduled scaling based on predicted traffic patterns
- **Time-Based Scaling**: Define scaling schedules for known peak periods (Black Friday, product launches, marketing campaigns, business hours)
- **Historical Analysis**: Analyze past traffic patterns to predict future scaling needs (daily/weekly/seasonal patterns)
- **Pre-Scaling**: Scale up resources 15-30 minutes before predicted peak to avoid scaling delays
- **Multi-Timezone Support**: Configure scaling schedules for different timezones (global user base)

**Configuration Example:**
- **Business Hours**: Scale up at 8:00 AM, scale down at 6:00 PM (local timezone)
- **Peak Days**: Scale up before Black Friday (00:00 UTC), maintain high capacity for 48 hours
- **Product Launches**: Scale up 1 hour before launch event, maintain for 4 hours post-launch
- **Seasonal Patterns**: Scale up during holiday seasons based on historical data

**Integration**: KEDA CronScaler triggers scaling based on cron expressions, works with HPA/KEDA for automatic scaling
- **Monitoring**: Track prediction accuracy (actual vs predicted traffic), adjust schedules based on accuracy
- **Fallback**: Combine with reactive scaling (KEDA/HPA) for unexpected traffic beyond predictions

---

## 3. Auto-Scaling Strategy

### 3.1 Pod Scaling

**Decision: KEDA + HPA for Pod Autoscaling**

**Implementation**: [KEDA](https://keda.sh/) for event-driven scaling, HPA for CPU/memory-based scaling
- **KEDA**: Queue depth (Azure Service Bus, RabbitMQ), custom metrics (request rate, active users), external triggers
- **HPA**: CPU/memory utilization (target: 70% CPU, 80% memory)
- **Scaling Config**: Min replicas based on 80% utilization, max replicas based on peak traffic, 5-minute scale-down cooldown, aggressive scale-up (2-3 minutes)

### 3.2 Node Autoscaling

**Decision: Managed Karpenter (Node Auto Provisioning) for Node Autoscaling**

**Implementation**: [Node Autoprovisioning](https://learn.microsoft.com/azure/aks/concepts-scale#node-autoprovisioning) (NAP) using Karpenter
- **Fast Provisioning**: Nodes provisioned in 30 seconds (vs minutes with Cluster Autoscaler)
- **Node Selection**: Auto-selects optimal VM SKU based on pod requirements (GPU, memory-optimized, CPU-optimized, general purpose)
- **Spot Instances**: Supports spot instances for non-critical workloads (70% cost savings)
- **Behavior**: Aggressive scale-up (30 seconds), conservative scale-down (5-minute delay), automatic node consolidation

---

## 4. Stateless vs Stateful Services

### 4.1 Stateless Services

**Design**: No local state, all state in external services (databases, cache, object storage)
- **Session Management**: External session store (Redis) instead of in-memory sessions
- **File Storage**: Azure Blob Storage instead of local disk
- **Benefits**: Easy horizontal scaling, simple deployment (blue-green, canary), no data loss on pod termination, cloud-agnostic

### 4.2 Stateful Services

**Implementation**: Managed services for production (PostgreSQL, MongoDB, Elasticsearch, Redis), StatefulSets in AKS for non-production only
- **Database State**: Managed services handle replication, backups, failover
- **File State**: Azure Blob Storage with geo-redundancy
- **Session State**: Redis with persistence and replication

---

## 5. Deployment Strategies

### 5.1 Blue-Green Deployment

**Decision: Blue-Green for Critical Services**

**Implementation**: ArgoCD manages blue/green environments, Azure Front Door/Application Gateway switches traffic instantly
- **Use Cases**: Critical production services, database schema migrations, major version upgrades
- **Trade-offs**: 2× resources during deployment (temporary), requires state management (database migrations, cache invalidation), longer deployment process

### 5.2 Canary Deployment

**Decision: Canary for Gradual Rollouts**

**Implementation**: Istio VirtualService manages traffic splitting (10% → 50% → 100%), ArgoCD controls progressive rollout
- **Traffic Split**: Phase 1 (10% for 15 min), Phase 2 (50% for 30 min), Phase 3 (100%)
- **Monitoring**: Real-time metrics (error rate, latency, CPU) with automatic rollback if threshold exceeded
- **Use Cases**: New feature releases, performance-sensitive services, risk mitigation
- **Trade-offs**: Requires Istio and monitoring, longer rollout process, minimal cost (no duplicate resources)

---

## 6. Database Scaling

### 6.1 Read Scaling

**PostgreSQL**: Read replicas (same-region and cross-region), PgBouncer connection pooling, application-level read/write splitting
**MongoDB (Cosmos DB)**: Multi-region reads (nearest region), autoscale throughput, session consistency
**Elasticsearch**: Index sharding, replica shards, optimized queries with caching

### 6.2 Write Scaling

**PostgreSQL**: PgBouncer connection pooling, batch writes, table partitioning (by date, tenant_id)
**MongoDB (Cosmos DB)**: Partition key (`user_id`, `tenant_id`), autoscale throughput, batch inserts

### 6.3 Database Failover

**Same-Region**: Automatic failover (RPO: < 1 minute, RTO: < 5 minutes), read replica promotion
**Multi-Region**: Manual failover to avoid split-brain (RPO: < 5 minutes, RTO: < 15 minutes), continuous replication, consistency validation

---

## 7. Rate Limiting and Throttling

### 7.1 API Rate Limiting

**Decision: Azure API Management (APIM) for API Rate Limiting**

**Strategy**: Per-user (1000 req/min authenticated), per-IP (100 req/min unauthenticated), per-service (SLA-based), burst allowance (2× normal rate for 5 seconds)
- **Throttling**: 429 Too Many Requests response, Retry-After header, quota exhaustion with upgrade path

### 7.2 Application-Level Throttling

**Circuit Breaker**: Istio circuit breaker (open after 5 consecutive failures, half-open after 30 seconds)
- **Backpressure**: KEDA scales based on queue depth, 30-second request timeout, graceful degradation (cached data or default responses)

---

## 8. Disaster Recovery (RPO/RTO)

### 8.1 Recovery Objectives

**Production**: RPO < 5 minutes, RTO < 15 minutes
**Non-Critical**: RPO < 1 hour, RTO < 4 hours

### 8.2 Multi-Cluster Active-Active Architecture

<p align="center">
  <img src="images/02-dc-dr-architecture.png" alt="Multi-Cluster Active-Active Disaster Recovery Architecture" width="800">
</p>

<p align="center">
  <em>Figure 4: Multi-Cluster Active-Active Disaster Recovery Architecture</em>
</p>

**Decision: Active-Active Multi-Region with Inter-Cluster Isolation**

**Architecture:**
- **Primary Cluster**: Germany West Central (Frankfurt) - Active (Primary)
- **Backup Cluster**: Germany North (Berlin) - Active (Backup)
- **Management Cluster**: Centralized ArgoCD and Azure Key Vault for GitOps and secrets management
- **Inter-Cluster Communication**: Prohibited between application pods (security boundary, prevents data inconsistency)

**Traffic Handling:**
- **Internet-Facing**: Azure Front Door with two Application Gateways (one per cluster) for global load balancing and automatic failover
- **Internal Traffic**: Azure Load Balancer distributes traffic to both clusters for internal endpoints
- **Failover Mechanism**: When a cluster fails, Application Gateway health checks become unhealthy, Azure Front Door automatically isolates the failed cluster and routes all traffic to the healthy cluster

**Pod Scaling During Failover:**
- **Emergency Mode**: Custom KEDA ScaledObject (cKEDA) watches emergency flag in ConfigMap
- **Automatic Scaling**: When emergency flag is `true`, autoscaler doubles the number of replicas to handle increased traffic from failed cluster
- **Scaling Triggers**: CPU utilization (70%), request rate, cron-based scheduling (business hours, peak periods)
- **Configuration**: Min/max replicas adjusted dynamically based on cluster health status

**Singleton Resources Management:**
- **Definition**: Resources that must operate exclusively on a single cluster (CronJobs, Debezium, StatefulSets with 1 replica, ArgoCronWorkflow)
- **Deployment**: ArgoCD deploys singleton resources only to primary cluster using Kustomize overlays
- **Active Cluster**: Resources operate normally
- **Non-Active Cluster**: Singleton resources deactivated (CronJobs suspended, Deployments set to `replicas: 0`)
- **Failover**: Infrastructure team manually migrates singleton resources to backup cluster after traffic recovery

**GitOps Deployment:**
- **ArgoCD**: Applies identical Kubernetes manifests to both active-active clusters simultaneously
- **Manifest Structure**: `base/` (common manifests), `environments/` (env-specific configs), `clusters/` (cluster-specific configs managed by Infrastructure Team)
- **Developer Restrictions**: Developers can only modify `base/` and `environments/`, Infrastructure Team manages `clusters/` via CODEOWNERS pattern

**Database Replication:**
- **PostgreSQL**: Cross-region replicas, PITR (7-day retention)
- **MongoDB (Cosmos DB)**: Multi-region writes, session consistency
- **Elasticsearch**: Cross-cluster replication
- **Redis**: Geo-replication
- **Backup**: Daily backups (30-day retention), geo-redundant storage in paired region

---

## 9. Failure and Chaos Scenarios

### 9.1 Failure Scenarios

**Single Pod**: Kubernetes auto-restart, health checks (liveness/readiness probes), load balancer routes away from unhealthy pods
**Node Failure**: Pod eviction to healthy nodes, Karpenter auto-provisions replacement, no service interruption (multi-node deployment)
**Region Failure**: Azure Front Door automatic failover, manual database failover (avoid split-brain), service restoration within RTO

### 9.2 Chaos Engineering

**Tools**: Chaos Mesh (Kubernetes-native), Litmus (open-source), custom Azure-specific scripts
- **Scenarios**: Pod termination, node draining, network partitions/latency, database failures/slow queries
- **Frequency**: Weekly (non-production), monthly (production during maintenance), before major releases

---

---

## 10. Required Questions

### 10.1 What is the first bottleneck you expect when traffic increases significantly?

**First Bottleneck: Database Connection Pool** - Application pods scale quickly (2-3 minutes), but connection pool is limited (each pod requires connections, pool has max limit)
- **Mitigation**: PgBouncer connection pooling, read replicas (offload read queries), optimize pool size (min/max per pod), database scaling (vertical/horizontal)

**Second Bottleneck: Application Pod Startup Time** - Pods take 30-60 seconds to start (container pull, health checks), new pods still starting during spike
- **Mitigation**: Pre-warmed pods (min replicas from historical data), optimize health checks (faster readiness), image caching, aggressive HPA scaling

**Third Bottleneck: Node Provisioning** - New nodes take 2-5 minutes (Karpenter reduces to 30 seconds, still bottleneck during extreme spikes)
- **Mitigation**: Karpenter fast provisioning (30 seconds), node pre-provisioning (buffer nodes), spot instances for faster provisioning

### 10.2 Which metrics do you use to measure system reliability?

**Metric Categories:**

**Work Metrics**: Measures how much useful output your system is producing (queries answered, pages served, requests processed, revenue generated). Used for availability management.

**Resource Metrics**: Measures how much of something is consumed to produce work (CPU, memory, disk, network). Used for capacity planning.

**Work Metrics:**

| Component | Metrics |
|-----------|---------|
| Front Door/WAF | Request count, 5XX error rate, response time |
| Application Gateway | 5XX rate, target 5XX rate, request count, response time |
| Application | Pod status |
| Database | Deadlocks, aborted clients, database connections, DML latency |
| SSL/TLS | Certificate expiration (synthetics API test) |

**Resource Metrics:**

| Component | Metrics |
|-----------|---------|
| Application Gateway | Active connections, new connection count, 2xx/3xx/4xx/5xx, target 2xx/3xx/4xx/5xx |
| Application | Container restart count, disk I/O, network I/O |
| Database | IOPS |

---


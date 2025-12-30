# Ecosystem Architecture

## System Diagram

Igor's product ecosystem is designed as an interconnected system where each product feeds insights and data into the others.

```
┌──────────────────────────────────────────────────────────────┐
│              Igor's Product Ecosystem v1.0                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐         ┌──────────────────┐            │
│  │  Audityzer   │────────▶│  Heatmap SaaS    │            │
│  │ (Security)   │         │   (Analytics)    │            │
│  └──────────────┘         └──────────────────┘            │
│         ▲                        ▲                          │
│         │                        │                          │
│    Insights              Events & Metrics                  │
│         │                        │                          │
│  ┌──────────────┐     ┌──────────────────┐               │
│  │ Structurizer │────▶│  Civic/GovTools  │               │
│  │(Architecture)│     │   (Governance)   │               │
│  └──────────────┘     └──────────────────┘               │
│                                                              │
│  ┌──────────────────────────────────────────────┐          │
│  │  Shared Infrastructure Layer                  │          │
│  │  Cloudflare, PostgreSQL, LLM, Auth, Billing  │          │
│  └──────────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

## Data Flow & Synergies

### 1. Audityzer → Heatmap
- **Input**: Security vulnerabilities detected
- **Processing**: Audityzer sends vulnerability metadata to Heatmap
- **Output**: Security dashboard showing threat trends over time

### 2. Heatmap → Civic
- **Input**: Aggregated events and metrics
- **Processing**: Heatmap normalizes all events
- **Output**: Government transparency dashboards, citizen engagement metrics

### 3. Structurizer → All
- **Input**: Architecture patterns
- **Processing**: Generates consistent boilerplate
- **Output**: All 4 products use shared patterns (LLM integrations, API structure, auth flow)

### 4. Civic → Audityzer
- **Input**: Public datasets
- **Processing**: Civic publishes government contracts and data
- **Output**: Audityzer can analyze government smart contracts for security

## Shared Infrastructure Layer

| Component | Purpose | Used By | Details |
|-----------|---------|---------|----------|
| **Cloudflare Workers** | API gateway, rate limiting, caching | All 4 | Global latency <50ms, auto-scaling |
| **PostgreSQL** | Primary data store | All 4 | Replicated across 3 regions |
| **Redis** | Cache + queue | All 4 | Sub-100ms response times |
| **LLM Layer** | AI reasoning & generation | Audityzer, Structurizer | GPT-4, Claude API + custom fine-tuning |
| **Auth System** | Role-based access control | All 4 | OAuth2 + JWT, GitHub + Email login |
| **Billing Engine** | Metering & payments | Heatmap, Structurizer | Stripe + custom metering |
| **Observability** | Monitoring & logging | All 4 | Prometheus, Grafana, DataDog |

## Deployment Architecture

### Edge (Cloudflare Workers)
```
Incoming Request
  ↓
Cloudflare Worker (rate limit, auth, route)
  ↓
Load balance to Region (US/EU/APAC)
  ↓
k8s cluster (Audityzer services)
  ↓
PostgreSQL (primary reads/writes)
  ↓
Redis (cache)
  ↓
Response (via Cloudflare Edge)
```

### Batch Processing
```
Heatmap Events → Kafka Queue
  ↓
Celery Workers (process, aggregate)
  ↓
PostgreSQL (store aggregates)
  ↓
Redis (cache for dashboard)
  ↓
API response
```

## Scalability & Reliability

- **Audityzer**: 5K contracts/day → 50M contracts/year
- **Heatmap**: 1M events/hour → 8.76B events/year
- **Structurizer**: 1K projects/month → 12K projects/year
- **Civic**: 100K citizens → 1M citizens

**99.9% Uptime SLA** across all products

## Next Steps (Q1 2025)

- [ ] GraphQL layer for unified queries
- [ ] Event streaming (Kafka → real-time websockets)
- [ ] Machine learning pipeline for anomaly detection
- [ ] Government API integrations (CKAN, OpenData)

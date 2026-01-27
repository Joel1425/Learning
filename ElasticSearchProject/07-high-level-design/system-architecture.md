# System Architecture

## Overview

This document describes the complete system architecture for the workspace provisioning service with Elasticsearch-backed dashboard filtering.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SYSTEM ARCHITECTURE                            │
└─────────────────────────────────────────────────────────────────────────────┘

                                   ┌─────────────────┐
                                   │   Client/User   │
                                   │     (CLI)       │
                                   └────────┬────────┘
                                            │
                                   REST API Calls
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            API GATEWAY LAYER                                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    AWS API Gateway + Load Balancer                    │ │
│  │                                                                       │ │
│  │  • Rate limiting          • Request routing                          │ │
│  │  • Authentication         • SSL termination                          │ │
│  │  • Request validation     • Load distribution                        │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
                    ▼                       ▼                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            APPLICATION LAYER                                │
│                                                                             │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │   A4C Service   │   │   A4C Service   │   │   A4C Service   │          │
│  │   (Instance 1)  │   │   (Instance 2)  │   │   (Instance 3)  │          │
│  │                 │   │                 │   │                 │          │
│  │ • createWS()    │   │ • createWS()    │   │ • createWS()    │          │
│  │ • getWS()       │   │ • getWS()       │   │ • getWS()       │          │
│  │ • deleteWS()    │   │ • deleteWS()    │   │ • deleteWS()    │          │
│  │ • scheduleImage │   │ • scheduleImage │   │ • scheduleImage │          │
│  │ • searchJobs()  │   │ • searchJobs()  │   │ • searchJobs()  │          │
│  └────────┬────────┘   └────────┬────────┘   └────────┬────────┘          │
│           │                     │                     │                    │
│           └─────────────────────┼─────────────────────┘                    │
│                                 │                                          │
│  ┌─────────────────┐           │           ┌─────────────────┐            │
│  │  Disk Service   │◀──────────┼──────────▶│  Disk Service   │            │
│  │  (Instance 1)   │           │           │  (Instance 2)   │            │
│  │                 │           │           │                 │            │
│  │ • getDiskUsage  │           │           │ • getDiskUsage  │            │
│  │ • findBestHost  │           │           │ • findBestHost  │            │
│  └─────────────────┘           │           └─────────────────┘            │
│                                 │                                          │
└─────────────────────────────────┼──────────────────────────────────────────┘
                                  │
        ┌─────────────────────────┴─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐     ┌───────────────────┐     ┌───────────────────┐
│   MESSAGING   │     │   DATA LAYER      │     │   SEARCH LAYER    │
│               │     │                   │     │                   │
│ ┌───────────┐ │     │ ┌───────────────┐ │     │ ┌───────────────┐ │
│ │   Kafka   │ │     │ │  PostgreSQL   │ │     │ │ Elasticsearch │ │
│ │           │ │     │ │   Primary     │ │     │ │    Cluster    │ │
│ │ Topics:   │ │     │ │               │ │     │ │               │ │
│ │ • job-sync│ │     │ │ Tables:       │ │     │ │ Index: jobs   │ │
│ │ • events  │ │     │ │ • users       │ │     │ │ Shards: 3     │ │
│ │           │ │     │ │ • workspace   │ │     │ │ Replicas: 1   │ │
│ └───────────┘ │     │ │ • job         │ │     │ │               │ │
│       │       │     │ │ • image       │ │     │ │ ┌───┐ ┌───┐   │ │
│       │       │     │ │ • schedule    │ │     │ │ │N1 │ │N2 │   │ │
│       ▼       │     │ └───────┬───────┘ │     │ │ └───┘ └───┘   │ │
│ ┌───────────┐ │     │         │         │     │ │    ┌───┐      │ │
│ │ ES Sync   │ │     │   WAL Replication │     │ │    │N3 │      │ │
│ │ Consumer  │─┼─────┼─────────┴─────────┼────▶│ │    └───┘      │ │
│ └───────────┘ │     │         │         │     │ └───────────────┘ │
│               │     │ ┌───────┴───────┐ │     │                   │
└───────────────┘     │ │   Replicas    │ │     └───────────────────┘
                      │ │  ┌───┐ ┌───┐  │ │
                      │ │  │R1 │ │R2 │  │ │
                      │ │  └───┘ └───┘  │ │
                      │ └───────────────┘ │
                      └───────────────────┘
                                  │
                                  │ SSH to Docker Daemon
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          COMPUTE LAYER                                      │
│                       (Bare Metal Cluster)                                  │
│                                                                             │
│  ┌─────────────────────────┐      ┌─────────────────────────┐              │
│  │      Server 1           │      │      Server 2           │              │
│  │                         │      │                         │              │
│  │  • Docker daemon        │      │  • Docker daemon        │              │
│  │  • Image builder daemon │      │  • Image builder daemon │              │
│  │  • Cleanup daemon       │      │  • Cleanup daemon       │              │
│  │                         │      │                         │              │
│  │  ┌───────┐ ┌───────┐   │      │  ┌───────┐ ┌───────┐   │              │
│  │  │  WS1  │ │  WS2  │   │      │  │  WS3  │ │  WS4  │   │              │
│  │  └───────┘ └───────┘   │      │  └───────┘ └───────┘   │              │
│  └─────────────────────────┘      └─────────────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                  ▲
                                  │ Polling for scheduled builds
                                  │
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EXTERNAL INTEGRATIONS                              │
│                                                                             │
│  ┌─────────────────┐                                                        │
│  │    CI System    │────────────▶ A4C Consumer ────────▶ A4C Service       │
│  │   (Jenkins,     │              (Kafka consumer)                          │
│  │    GitHub, etc) │                                                        │
│  └─────────────────┘                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Layer Descriptions

### 1. Client Layer
- CLI tool for workspace management
- Web dashboard for job monitoring
- REST API consumers

### 2. API Gateway Layer
- **AWS API Gateway**: Request routing, rate limiting
- **Load Balancer**: Distributes traffic across service instances
- Handles authentication, SSL termination

### 3. Application Layer
- **A4C Service**: Core workspace provisioning service
  - Multiple instances on Kubernetes
  - Handles CRUD operations
  - Queries Elasticsearch for dashboard
- **Disk Service**: Finds optimal server for workspace
  - Returns server with least disk usage

### 4. Data Layer
- **PostgreSQL Primary**: Source of truth for all data
  - ACID transactions
  - Handles all writes
- **PostgreSQL Replicas**: Read scaling
  - Async WAL replication
  - Used for fallback queries

### 5. Search Layer
- **Elasticsearch Cluster**: Optimized for read queries
  - 3-node cluster
  - Dashboard filtering
  - Full-text search on errors

### 6. Messaging Layer
- **Kafka**: Event streaming
  - Decouples services
  - Reliable message delivery
- **ES Sync Consumer**: Syncs PostgreSQL to Elasticsearch

### 7. Compute Layer
- **Bare Metal Servers**: Run actual workspaces
  - Docker containers
  - Image builder daemons
  - Cleanup daemons

---

## Data Flow Summary

| Flow | Path | Purpose |
|------|------|---------|
| Write | Client → Gateway → A4C → PostgreSQL → Kafka → ES | Create/update data |
| Read (Dashboard) | Client → Gateway → A4C → Elasticsearch | Fast filtering |
| Read (Fallback) | Client → Gateway → A4C → PostgreSQL Replica | When ES down |
| Workspace Create | A4C → Disk Service → Docker Daemon | Provision container |
| Image Build | CI System → A4C Consumer → A4C → Bare Metal | Build Docker images |

---

## Technology Stack

| Layer | Technology | Version |
|-------|------------|---------|
| Language | Java | 17 |
| Framework | Spring Boot | 3.x |
| API Gateway | AWS API Gateway | - |
| Load Balancer | AWS ALB | - |
| Container Orchestration | Kubernetes | 1.28 |
| Database | PostgreSQL | 15 |
| Search | Elasticsearch | 8.x |
| Messaging | Apache Kafka | 3.x |
| Container Runtime | Docker | 24.x |
| Monitoring | Prometheus + Grafana | - |
| Logging | ELK Stack | - |

---

## Interview Talking Point

> "Our architecture separates concerns clearly. PostgreSQL is the source of truth for all data. Elasticsearch provides fast read access for the dashboard. Kafka decouples the sync process. Multiple service instances run on Kubernetes for high availability. The bare metal cluster runs actual workspaces with their own daemon processes."

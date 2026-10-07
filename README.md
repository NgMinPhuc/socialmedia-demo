<div align="center">

# Community-Centric Platform

**A Reddit + Discord style community platform built as polyglot microservices (Java + Go).**

<p>
  <img src="https://img.shields.io/badge/Java-17-ED8B00?logo=openjdk&logoColor=white" alt="Java 17"/>
  <img src="https://img.shields.io/badge/Spring_Boot-3.4-6DB33F?logo=springboot&logoColor=white" alt="Spring Boot"/>
  <img src="https://img.shields.io/badge/Go-1.23-00ADD8?logo=go&logoColor=white" alt="Go"/>
  <img src="https://img.shields.io/badge/gRPC-1.63-244c5a?logo=grpc&logoColor=white" alt="gRPC"/>
  <img src="https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white" alt="PostgreSQL"/>
  <img src="https://img.shields.io/badge/MongoDB-7-47A248?logo=mongodb&logoColor=white" alt="MongoDB"/>
  <img src="https://img.shields.io/badge/RabbitMQ-3.13-FF6600?logo=rabbitmq&logoColor=white" alt="RabbitMQ"/>
  <img src="https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white" alt="Redis"/>
  <img src="https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black" alt="React"/>
</p>

**[🔗 Live Demo](https://tesoh.duckdns.org/)** · **[🏛️ Architecture](./docs/ARCHITECTURE.md)** · **[🗄️ Database](./docs/DATABASE.md)** · **[🌐 API](./docs/API.md)**

![Demo](./docs/images/demo.png)

</div>

> **Note:** This repository contains the public documentation for the project. The source code is kept in a private repository.

---

## What is it?

A social platform centered on **communities** rather than personal feeds. It combines two well-known models:

| From Reddit | From Discord |
|---|---|
| Threads inside forum channels | Communities (servers) with categories & channels |
| Unlimited-depth nested comments | Real-time text channels |
| Upvote / downvote, Hot / Top / New ranking | Roles with bitmask permissions, private channels |
| User karma | Direct messages & group DMs, typing indicators, presence |

There is intentionally **no Facebook/Instagram-style personal feed** — content lives inside communities.

## Demo

| | |
|---|---|
| **Live demo** | https://tesoh.duckdns.org/ |
| **Demo account** | `demo@example.com` / `demo-password` |

| Community & channels | Forum threads & nested comments |
|---|---|
| ![Community](./docs/images/screenshot-community.png) | ![Discussion](./docs/images/screenshot-discussion.png) |
| **Real-time chat / DMs** | **Explore & search** |
| ![Chat](./docs/images/screenshot-chat.png) | ![Explore](./docs/images/screenshot-explore.png) |

## Key Features

- **Authentication** — register/login with short-lived JWT access tokens, rotating refresh tokens, and Redis-backed logout (token blacklist). Global RBAC for staff/admin.
- **Profiles & social graph** — profiles, karma, follow / friend / block relationships modeled directly in PostgreSQL.
- **Communities** — create/join (public or invite code), categories, text and forum channels, custom roles with Discord-style permission bitmasks, per-channel role access.
- **Discussions** — threads with markdown and attachments, nested comment trees (PostgreSQL `ltree`), voting, Hot/Top/New sorting.
- **Messaging** — channel chat and DMs / group DMs stored in MongoDB, with cursor-based infinite scroll, mentions, unread counts.
- **Real-time** — a single WebSocket per client for messages, typing, presence and notifications.
- **Media** — direct-to-storage uploads via presigned URLs (MinIO / S3 compatible).
- **Search** — full-text search of users, communities and threads with Meilisearch.
- **Background processing** — karma calculation, notifications and counters handled asynchronously by Go workers.
- **Admin dashboard** — account management for staff roles.

## Architecture at a Glance

![System Architecture](./docs/images/architecture.png)

```mermaid
flowchart LR
    Client["React SPA"] -->|REST| GW["Traefik<br/>API Gateway"]
    Client <-->|WebSocket| RT["Realtime Gateway (Go)"]

    GW --> ID["Identity (Java)"]
    GW --> US["User (Java)"]
    GW --> CO["Community (Java)"]
    GW --> DI["Discussion (Java)"]
    GW --> MS["Messaging (Go)"]
    GW --> ME["Media (Go)"]

    DI -. gRPC .-> CO
    DI -. gRPC .-> US
    MS -. gRPC .-> CO
    MS -. gRPC .-> US
    MS -. gRPC .-> RT
    RT -. gRPC .-> MS

    ID & CO & DI & MS --> MQ[["RabbitMQ"]]
    MQ --> WK["Background Workers (Go)"]
    MQ --> US
    WK -. gRPC .-> RT
```

- **8 services**, each owning its own database (**database-per-service**).
- **gRPC** for synchronous service-to-service calls, **RabbitMQ** for domain events.
- **One WebSocket** connection per client to the Realtime Gateway; backends push live events through it via gRPC.

Read the full design in **[ARCHITECTURE.md](./docs/ARCHITECTURE.md)**.

## Tech Stack

| Area | Technology |
|---|---|
| Java services | Java 17, Spring Boot 3.4, Spring Security 6, Spring Data JPA, Flyway |
| Go services | Go 1.23, Gin, gorilla/websocket, mongo-driver |
| Communication | gRPC + Protocol Buffers (sync), RabbitMQ topic exchange (async) |
| Gateway | Traefik v3 |
| Databases | PostgreSQL 15 (one DB per service), MongoDB 7 |
| Cache / realtime state | Redis 7 |
| Search | Meilisearch |
| Object storage | MinIO (S3 compatible) |
| Frontend | React 18, TypeScript, Vite, TailwindCSS, Zustand, TanStack Query, React Hook Form + Zod |
| Infrastructure | Docker, Docker Compose |
| Testing | JUnit 5, Mockito, Spring Boot Test, Go `testing` |

## Services

| Service | Language | Responsibility | Storage |
|---|---|---|---|
| `identity-service` | Java | Accounts, login, JWT, refresh tokens, global roles | PostgreSQL, Redis |
| `user-service` | Java | Profiles, karma, social graph, settings | PostgreSQL, Meilisearch |
| `community-service` | Java | Communities, channels, members, roles, permissions | PostgreSQL, Redis, Meilisearch |
| `discussion-service` | Java | Threads, nested comments, votes, ranking | PostgreSQL, Redis, Meilisearch |
| `messaging-service` | Go | Channel messages, DMs, conversations | MongoDB, Redis |
| `media-service` | Go | Upload / download of files via presigned URLs | MongoDB, MinIO |
| `realtime-gateway` | Go | WebSocket connections, fan-out, presence | Redis |
| `background-workers` | Go | Karma, notifications, counters (event consumers) | Redis |

## Documentation

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](./docs/ARCHITECTURE.md) | System design, service boundaries, communication patterns, key flows, security |
| [DATABASE.md](./docs/DATABASE.md) | ER diagrams of every database, MongoDB collections, Redis key design |
| [API.md](./docs/API.md) | REST endpoints, WebSocket events, internal gRPC methods |

## Author

**Your Name** — [GitHub](https://github.com/your-username) · [LinkedIn](https://linkedin.com/in/your-profile) · your-email@example.com

# Database Design

The platform follows **database-per-service**: each service owns its own database and no service queries another's. Cross-service references (e.g. `author_id` in a thread) are plain UUIDs, not foreign keys — consistency between services is kept through APIs and events.

```mermaid
flowchart LR
    ID["Identity"] --- A[("identity_db<br/>PostgreSQL")]
    US["User"] --- B[("user_db<br/>PostgreSQL")]
    CO["Community"] --- C[("community_db<br/>PostgreSQL")]
    DI["Discussion"] --- D[("discussion_db<br/>PostgreSQL + ltree")]
    MS["Messaging"] --- E[("messaging_db<br/>MongoDB")]
    ME["Media"] --- F[("media_db<br/>MongoDB")] & G[("MinIO")]
    ALL["Shared infra"] --- R[("Redis")] & M[("Meilisearch")]
```

## Conventions

- **Primary keys:** UUID (`uuid_generate_v4()`) in PostgreSQL; `ObjectId` in MongoDB. UUIDs avoid ID enumeration and ease future sharding.
- **Audit & soft delete:** every table/collection has `status`, `created_at`, `updated_at`, `deleted_at`. Reads always filter `deleted_at IS NULL`.
- **Partial indexes:** unique and lookup indexes are declared `WHERE deleted_at IS NULL`, so a soft-deleted row never blocks a new one (e.g. re-using a username).
- **Naming:** `snake_case` columns; Java/Go fields map to them.
- **Migrations:** versioned and applied on service startup.

---

## 1. Identity — `identity_db` (PostgreSQL)

Accounts, credentials, sessions and global role-based access control.

```mermaid
erDiagram
    accounts ||--o{ account_roles : has
    roles ||--o{ account_roles : "granted to"
    roles ||--o{ role_permissions : includes
    permissions ||--o{ role_permissions : "part of"
    accounts ||--o{ refresh_tokens : owns

    accounts {
        uuid id PK
        varchar email UK
        varchar username UK
        varchar password_hash "BCrypt"
        varchar status "ACTIVE, PENDING, SUSPENDED, BANNED"
        timestamptz email_verified_at
        timestamptz password_changed_at
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    roles {
        uuid id PK
        varchar code UK "ROLE_USER, ROLE_MODERATOR, ROLE_ADMIN"
        varchar description
        varchar status
    }
    permissions {
        uuid id PK
        varchar code UK
        varchar description
        varchar status
    }
    account_roles {
        uuid account_id PK, FK
        uuid role_id PK, FK
        varchar status
    }
    role_permissions {
        uuid role_id PK, FK
        uuid permission_id PK, FK
        varchar status
    }
    refresh_tokens {
        uuid id PK
        uuid account_id FK
        char token_hash UK "SHA-256, never the raw token"
        varchar user_agent
        varchar client_ip
        boolean is_revoked
        timestamptz expires_at
        timestamptz created_at
    }
```

**Notes**
- Email and username are unique case-insensitively (`lower(...)`) among non-deleted accounts.
- Only a hash of each refresh token is stored; tokens are rotated on every refresh. `password_changed_at` lets services reject tokens issued before a password change.

---

## 2. User — `user_db` (PostgreSQL)

Public profiles, karma, social graph and user preferences. A profile shares its id with the identity account and is created from the `user.registered` event.

```mermaid
erDiagram
    user_profiles ||--o{ user_relationships : "initiates"
    user_profiles ||--o{ user_relationships : "is target of"
    user_profiles ||--|| user_settings : has

    user_profiles {
        uuid user_id PK "same id as identity account"
        varchar username UK
        varchar display_name
        text bio
        varchar avatar_url
        varchar banner_url
        bigint karma_points
        varchar status "ACTIVE, PRIVATE, DEACTIVATED, BANNED"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    user_relationships {
        uuid user_id PK, FK
        uuid target_user_id PK, FK
        varchar relationship_type PK "FOLLOW, FRIEND, BLOCK"
        varchar status "PENDING, ACCEPTED"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    user_settings {
        uuid user_id PK, FK
        boolean allow_direct_messages
        boolean notify_on_mention
        varchar theme "DARK, LIGHT"
        jsonb custom_preferences
        timestamptz updated_at
    }
    processed_karma_batches {
        uuid id PK "idempotency key for karma events"
        timestamptz created_at
    }
```

**Notes**
- **Social graph in PostgreSQL:** composite PK `(user_id, target_user_id, relationship_type)` plus indexes in both directions make "who do I follow", "who follows me" and "is X blocked" index-only lookups — no graph database required.
- `processed_karma_batches` records already-applied karma batches so redelivered events are ignored.

---

## 3. Community — `community_db` (PostgreSQL)

Discord-style structure: communities, categories, channels, members, roles and channel access.

```mermaid
erDiagram
    communities ||--o{ categories : contains
    communities ||--o{ channels : contains
    categories |o--o{ channels : groups
    communities ||--o{ community_roles : defines
    communities ||--o{ community_members : has
    community_members ||--o{ member_roles : holds
    community_roles ||--o{ member_roles : "assigned via"
    channels ||--o{ channel_role_access : "restricted by"
    community_roles ||--o{ channel_role_access : "grants"

    communities {
        uuid id PK
        varchar name
        varchar slug UK
        text description
        varchar icon_url
        varchar banner_url
        uuid owner_id "user id (no FK)"
        boolean is_private
        varchar invite_code UK
        int member_count
        varchar status "ACTIVE, ARCHIVED, BANNED"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    categories {
        uuid id PK
        uuid community_id FK
        varchar name
        int position
        varchar status
    }
    channels {
        uuid id PK
        uuid community_id FK
        uuid category_id FK "nullable"
        varchar name
        varchar type "TEXT_CHAT, THREAD_FORUM"
        text topic
        int position
        boolean is_private
        varchar status
    }
    community_roles {
        uuid id PK
        uuid community_id FK
        varchar name
        varchar color_hex
        int position
        bigint permissions_bitmask
        boolean is_default "the @everyone role"
        varchar status
    }
    community_members {
        uuid id PK
        uuid community_id FK
        uuid user_id "user id (no FK)"
        varchar nickname
        varchar status "ACTIVE, MUTED, KICKED, BANNED"
        timestamptz joined_at
    }
    member_roles {
        uuid member_id PK, FK
        uuid role_id PK, FK
        varchar status
    }
    channel_role_access {
        uuid channel_id PK, FK
        uuid role_id PK, FK
        varchar status
    }
```

**Notes**
- **Permissions as a bitmask:** each role stores a 64-bit mask (view, send messages, create threads, manage channels, manage roles…). A member's effective permissions are the OR of all their roles; the result is cached in Redis.
- **Private channels:** visible only to roles listed in `channel_role_access`.
- `(community_id, user_id)` is unique among active members. A discovery index on `(is_private, status, member_count DESC)` powers the Explore page.

---

## 4. Discussion — `discussion_db` (PostgreSQL + `ltree`)

Reddit-style threads, nested comments and votes.

```mermaid
erDiagram
    threads ||--o{ comments : has
    comments |o--o{ comments : "replies to"
    threads ||--o{ thread_votes : receives
    comments ||--o{ comment_votes : receives

    threads {
        uuid id PK
        uuid community_id "no FK (other service)"
        uuid channel_id "no FK (other service)"
        uuid author_id "no FK (other service)"
        varchar title
        text content_markdown
        jsonb attachments
        text_array tags
        int upvotes
        int downvotes
        int score
        int comment_count
        boolean is_pinned
        boolean is_locked
        varchar status "ACTIVE, LOCKED, REMOVED, ARCHIVED"
        timestamptz created_at
        timestamptz updated_at
        timestamptz deleted_at
    }
    comments {
        uuid id PK
        uuid thread_id FK
        uuid parent_comment_id FK "nullable"
        uuid author_id
        ltree path "e.g. root.c1.c4"
        int depth
        text content
        int upvotes
        int downvotes
        int score
        varchar status "ACTIVE, DELETED_BY_USER, REMOVED_BY_MOD"
        timestamptz created_at
        timestamptz deleted_at
    }
    thread_votes {
        uuid thread_id PK, FK
        uuid user_id PK
        smallint vote_type "+1 or -1"
        timestamptz updated_at
    }
    comment_votes {
        uuid comment_id PK, FK
        uuid user_id PK
        smallint vote_type "+1 or -1"
        timestamptz updated_at
    }
```

**Notes**
- **Comment trees with `ltree`:** every comment stores its full ancestor path. A GiST index on `path` lets the service fetch a whole thread's tree, or any subtree (`path <@ 'root.c1'`), in one query, already ordered for rendering.
- Deleted comments are soft-deleted so replies under them remain intact (shown as "[deleted]").
- **One vote per user per target** is guaranteed by the composite primary key; changing a vote is an upsert, and counters are updated in the same transaction.
- A composite index on `(community_id, channel_id, score DESC, created_at DESC)` serves Top/New listings; Hot ranking is served from a Redis sorted set.

---

## 5. Messaging — `messaging_db` (MongoDB)

Channel chat and direct messages. MongoDB fits the append-heavy, time-ordered access pattern.

```mermaid
erDiagram
    channel_messages {
        ObjectId _id PK
        string channel_id "community channel (uuid)"
        string sender_id
        string content
        Attachment[] attachments
        string reply_to_message_id
        string[] mentions
        string client_msg_id "client-side dedupe"
        bool is_pinned
        string status "ACTIVE, EDITED, DELETED"
        date created_at
        date updated_at
        date deleted_at
    }
    conversations ||--o{ direct_messages : contains
    conversations {
        ObjectId _id PK
        string type "DIRECT, GROUP_DM"
        string pair_key "unique for 1-1 DMs"
        string name "group DMs only"
        Participant[] participants "user_id + ACTIVE/LEFT"
        LastMessage last_message "denormalized preview"
        map last_read_message_id "per user"
        string status
        date created_at
        date updated_at
        date deleted_at
    }
    direct_messages {
        ObjectId _id PK
        ObjectId conversation_id FK
        string sender_id
        string content
        Attachment[] attachments
        string reply_to_message_id
        string client_msg_id
        string status "ACTIVE, EDITED, DELETED"
        date created_at
        date deleted_at
    }
```

**Indexes**

| Collection | Index | Used for |
|---|---|---|
| `channel_messages` | `{ channel_id: 1, created_at: -1 }` | Channel history, cursor pagination |
| `channel_messages` | `{ channel_id: 1, is_pinned: 1 }` | Pinned messages |
| `conversations` | `{ "participants.user_id": 1, updated_at: -1 }` | Inbox ordered by latest activity |
| `conversations` | `{ pair_key: 1 }` (unique, partial) | At most one 1-1 DM per pair of users |
| `direct_messages` | `{ conversation_id: 1, created_at: -1 }` | DM history |

`client_msg_id` makes sending idempotent: a client retry after a network drop never creates a duplicate message.

---

## 6. Media — `media_db` (MongoDB + MinIO)

File metadata lives in MongoDB; the bytes live in MinIO (S3 compatible).

```mermaid
erDiagram
    media {
        ObjectId _id PK
        string owner_id
        string purpose "AVATAR, BANNER, THREAD, MESSAGE..."
        string bucket
        string object_key "final location"
        string staging_key "presigned upload target"
        string content_type
        long size_bytes
        string original_name
        string url
        string status "PENDING, ACTIVE, DELETED"
        date created_at
        date updated_at
        date deleted_at
    }
```

Lifecycle: `PENDING` (presigned URL issued) → `ACTIVE` (owning service confirmed it) → `DELETED`. Unconfirmed uploads can be cleaned up from staging.

---

## 7. Redis Key Design

One Redis instance, namespaced by service prefix.

| Owner | Key pattern | Type | TTL | Purpose |
|---|---|---|---|---|
| Identity | `auth:blacklist:{token_hash}` | String | token's remaining life | Revoked access tokens (logout) |
| Identity | `auth:login_fails:{email}` | Counter | 15 min | Brute-force protection |
| Community | `comm:perms:{community_id}:{user_id}` | String (int64 mask) | 30 min | Cached effective permissions |
| Discussion | `disc:hot:{channel_id}` | Sorted set | 24 h | Hot/Trending ranking |
| Messaging | `msg:typing:{channel_id}:{user_id}` | String | 3–5 s | Typing indicator |
| Messaging | `msg:unread:{channel_id}:{user_id}` | Counter | 7 days | Unread counts |
| Realtime | `rt:presence:{user_id}` | String | 60 s (heartbeat) | Online / idle status |
| Realtime | `rt:socket_node:{user_id}` | String | 120 s | Which gateway node holds the user's socket |
| Realtime | `rt:pubsub:channel:{channel_id}` | Pub/Sub | — | Fan-out across gateway nodes |

Eviction policy is LRU, so Redis degrades gracefully under memory pressure — it is only ever a cache or ephemeral state, never the source of truth.

---

## 8. Search Indexes (Meilisearch)

| Index | Owner | Fed from |
|---|---|---|
| `users` | User | Profile create / update |
| `communities` | Community | Public community create / update |
| `threads` | Discussion | Thread create / update / delete |

Each service keeps its own index in sync with its own database.

# API Overview

The platform exposes three kinds of interfaces:

| Interface | Audience | Transport |
|---|---|---|
| **REST** | Web client | HTTPS via Traefik, prefix `/api/v1/<domain>` |
| **WebSocket** | Web client | One connection to the Realtime Gateway |
| **gRPC** | Internal services only | Protocol Buffers, not exposed publicly |

---

## Conventions

**Response envelope** — every REST response uses the same shape:

```json
{
  "code": 200,
  "message": "Success",
  "result": { }
}
```

Errors use the same envelope with a domain error code and `result: null`, plus the matching HTTP status (400, 401, 403, 404, 409…). Validation errors list the offending fields. Stack traces are never returned.

**Authentication** — `Authorization: Bearer <accessToken>`. Access tokens live 15 minutes; refresh them with `/auth/refresh-token`.

**Pagination** — relational lists use `page` / `size`; chat history uses cursor pagination (`before=<messageId>`) for infinite scroll.

**Auth column legend** — 🔓 public · 🔐 signed-in · 👑 role/permission required.

---

## Identity — `/api/v1/auth`

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/register` | Create an account (email, username, password) | 🔓 |
| POST | `/login` | Returns `accessToken` + `refreshToken` | 🔓 |
| POST | `/refresh-token` | Rotate the refresh token, get a new access token | 🔓 |
| POST | `/logout` | Revoke the current tokens | 🔐 |
| POST | `/change-password` | Change password (invalidates older tokens) | 🔐 |

### Admin — `/api/v1/admin/accounts`

| Method | Path | Description | Auth |
|---|---|---|---|
| GET | `/` | Search / list accounts | 👑 staff |
| GET | `/stats` | Account statistics for the dashboard | 👑 staff |
| PUT | `/{id}/roles` | Change an account's global roles | 👑 admin |
| PUT | `/{id}/status` | Suspend / ban / reactivate an account | 👑 staff |

## User — `/api/v1/users`

| Method | Path | Description | Auth |
|---|---|---|---|
| GET | `/me` | My profile | 🔐 |
| PUT | `/me` | Update display name, bio, avatar, banner | 🔐 |
| GET | `/{username}` | Public profile with karma | 🔓 |
| GET | `/search?q=` | Search users | 🔓 |
| POST | `/{id}/relationship` | `FOLLOW`, `UNFOLLOW`, `FRIEND_REQUEST`, `BLOCK`… | 🔐 |
| GET | `/{id}/relationships` | Followers, following, friends, blocked | 🔐 |

## Community — `/api/v1/communities`

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/` | Create a community (creator becomes owner) | 🔐 |
| GET | `/me` | Communities I belong to (sidebar) | 🔐 |
| GET | `/search?q=` | Explore public communities | 🔓 |
| GET | `/{id}` | Community details with categories & channels | 🔐 |
| PUT | `/{id}` | Update name, icon, banner, description | 👑 owner/admin |
| POST | `/{id}/join` | Join (public, or with invite code) | 🔐 |
| DELETE | `/{id}/leave` | Leave a community | 🔐 |
| POST | `/{id}/channels` | Create a `TEXT_CHAT` or `THREAD_FORUM` channel | 👑 manage channels |
| PUT | `/{id}/channels/{channelId}` | Rename, reorder, change topic or visibility | 👑 manage channels |
| GET | `/{id}/members` | Member list (paged) | 🔐 |

## Discussion — `/api/v1/discussions`

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/threads` | Create a thread in a forum channel | 👑 create threads |
| GET | `/channels/{channelId}/threads?sort=HOT\|TOP\|NEW` | List threads in a channel | 🔓 |
| GET | `/threads/{id}` | Thread details | 🔓 |
| DELETE | `/threads/{id}` | Soft-delete a thread | 👑 author/mod |
| POST | `/threads/{id}/comments` | Comment (with optional `parentCommentId` to reply) | 🔐 |
| GET | `/threads/{id}/comments` | Full comment tree, ordered for rendering | 🔓 |
| DELETE | `/comments/{id}` | Soft-delete a comment (replies are kept) | 👑 author/mod |
| POST | `/votes` | Vote `+1` / `-1` on a `THREAD` or `COMMENT` | 🔐 |

## Messaging — `/api/v1/messages`

| Method | Path | Description | Auth |
|---|---|---|---|
| GET | `/channels/{channelId}?before=` | Channel history (cursor) | 🔐 channel access |
| DELETE | `/{id}` | Delete a channel message | 👑 author/mod |
| POST | `/conversations` | Start a DM or group DM | 🔐 |
| GET | `/conversations` | My inbox with last message & unread count | 🔐 |
| GET | `/conversations/{id}` | DM history | 🔐 participant |
| POST | `/conversations/{id}/messages` | Send a DM over REST (fallback to WebSocket) | 🔐 participant |
| DELETE | `/conversations/{id}/messages/{msgId}` | Delete a DM | 🔐 author |

## Media — `/api/v1/media`

| Method | Path | Description | Auth |
|---|---|---|---|
| POST | `/upload-url` | Get a presigned URL to upload a file directly to storage | 🔐 |
| GET | `/{id}` | File metadata and download URL | 🔐 |

---

## WebSocket — Realtime Gateway

**Connect:** `wss://<host>/ws?token=<accessToken>` — the token is validated before the connection is upgraded.

**Frame format** (both directions):

```json
{
  "event": "NEW_MESSAGE",
  "channelId": "optional-uuid",
  "payload": { }
}
```

### Client → Server

| Event | Description |
|---|---|
| `SUBSCRIBE_CHANNEL` | Start receiving a channel's live events (access is checked) |
| `UNSUBSCRIBE_CHANNEL` | Stop receiving a channel's events |
| `SEND_CHANNEL_MESSAGE` | Send a chat message to a channel |
| `SEND_DIRECT_MESSAGE` | Send a DM |
| `TYPING` | "User is typing…" signal (expires after a few seconds) |

### Server → Client

| Event | Description |
|---|---|
| `NEW_MESSAGE` | New message in a subscribed channel or DM |
| `MESSAGE_DELETED` | A message was removed |
| `USER_TYPING` | Someone is typing in the channel / DM |
| `USER_PRESENCE` | A friend went online / offline |
| `NOTIFICATION` | Mention, reply, or other notification |

Sent messages are acknowledged with the client's `clientMsgId`, so the UI can show optimistic messages and reconcile them safely.

---

## Internal gRPC (not public)

| Service | Methods |
|---|---|
| **User** | `GetUserProfile`, `GetBatchUsers`, `CheckUserRelation` |
| **Community** | `CheckPermission`, `VerifyChannelAccess`, `AuthorizeChannelSubscription` |
| **Messaging** | `SendChannelMessage`, `SendDirectMessage` |
| **Realtime** | `BroadcastToChannel`, `SendToUser` |
| **Media** | `GetMedia`, `ConfirmMedia`, `GetDownloadUrl`, `DeleteMedia` |

See [ARCHITECTURE.md](./ARCHITECTURE.md#41-synchronous--grpc) for who calls what and why.

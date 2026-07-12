# Mattermost Channel — Design & Specification

## Goal

Implement a first-class Mattermost channel (`nanobot/channels/mattermost.py`) as a **built-in** that matches the feature parity and UX polish of the existing Slack channel. A separate PyPI plugin (`nanobot-channel-mattermost`) is **not** the goal — this lives in-tree alongside Slack, Discord, Telegram, etc.

---

## Mattermost Server Requirements

The Mattermost server administrator must enable two settings before a bot can be created.

### Prerequisites Checklist

| Setting | Location (System Console) | Default |
|---------|--------------------------|---------|
| **Enable Bot Account Creation** | Integrations → Bot Accounts | `false` (off) |
| **Enable Personal Access Tokens** | Integrations → Integration Management | `false` (off) |

Both are **off by default** on a fresh Mattermost install — skipping either is the most common setup failure.

### Step-by-Step Setup

**1. Enable bot accounts and personal access tokens**

As a System Admin, navigate to:

- **System Console → Integrations → Bot Accounts** → set **Enable Bot Account Creation** to `true` → Save
- **System Console → Integrations → Integration Management** → set **Enable Personal Access Tokens** to `true` → Save

**2. Create a bot account**

Navigate to **Product menu → Integrations → Bot Accounts → Add Bot Account**:

| Field | Description |
|-------|-------------|
| **Username** | Must start with a letter, 3–22 lowercase alphanumeric chars, plus `.`, `-`, `_`. |
| **Display Name** | Optional human-readable name. |
| **Description** | Optional. |
| **Role** | `Member` (default) — sufficient for most setups. `System Admin` grants full access. |
| **Post to all channels** | Optional — lets the bot post to any channel without being a member. |
| **Post to all public channels** | Optional — lets the bot post to any public channel without being a member. |

Click **Create Bot Account**.

**3. Copy the bot token**

Immediately after creation, Mattermost displays the **access token** once in a banner. Copy it and store it securely — **it is never shown again**. This is the value used as `token` in the channel config.

> The **Token ID** shown in the bot list is not the access token. Only the one-time banner value works for API auth.

**4. Add the bot to channels**

The bot must be explicitly added to each channel (public or private) it interacts in:

- Select the team → **Invite People**
- Enter the bot's username
- Select **Invite Member**
- The bot now appears in the channel's member list

For DMs, any user can DM the bot by username — no channel membership required.

**5. Verify the token**

```bash
curl -s -H "Authorization: Bearer YOUR_TOKEN" \
  https://your-server/api/v4/users/me
```

Returns `200` with the bot's user object on success. Returns `401` if the token is invalid.

### Required Bot Permissions

The bot account needs these implicit permissions (all granted by the `Member` role by default in channels it belongs to):

| Permission | Needed For |
|-----------|------------|
| `read_channel` | Read messages in channels the bot is a member of |
| `create_post` | Send messages |
| `edit_post` | Edit messages (required for streaming) |
| `upload_file` | Upload file attachments |
| `add_reaction` | Add emoji reactions |
| `read_user` | Look up user profiles (identity resolution) |
| `get_public_link` | Download files (if `post:all` role or channel member) |

If the bot has the **Post to all channels** role, it also gets `post:all` / `post:channels` which allows posting without explicit channel membership.

### Bot Token vs Personal Access Token

There are two token types, both valid for the `token` config field:

| Token Type | Created By | Lifetime |
|-----------|-----------|----------|
| **Bot token** | Bot Accounts UI | Permanent (survives bot reactivation) |
| **Personal Access Token** (PAT) | Profile → Security → Personal Access Tokens | Permanent (revoked on account deactivation) |

A PAT can be used when you want nanobot to post as a regular user account rather than as a separate bot. Both authenticate the same way via `Authorization: Bearer <token>`.

---

## Architecture

```
  Mattermost WebSocket  ──►  _handle_websocket_message()
                                   │
                          _handle_posted_event()
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
         _handle_dm()      _handle_channel()    _handle_interactive()
              │                    │                    │
              └────────┬───────────┘                    │
                       │                                │
              _handle_message()  ◄──────────────────────┘
                       │
                 MessageBus
                       │
                 AgentLoop
                       │
                    send()
                       │
              Mattermost REST API
```

Inbound events arrive via **Mattermost WebSocket API** (`/api/v4/websocket`). Outbound messages are sent via **Mattermost REST API** (`/api/v4/posts`, `/api/v4/files`, etc.).

---

## Configuration

```json
{
  "channels": {
    "mattermost": {
      "enabled": true,
      "serverUrl": "https://your-mattermost.example.com",
      "token": "YOUR_MATTERMOST_BOT_TOKEN",
      "teamId": "",
      "allowFromMatchMode": "id",
      "allowFrom": ["YOUR_USER_ID"],
      "groupPolicy": "mention",
      "groupAllowFrom": [],
      "replyInThread": true,
      "includeThreadContext": true,
      "threadContextLimit": 20,
      "streaming": true,
      "streamingMaxChars": 16000,
      "reactEmoji": "eyes",
      "doneEmoji": "white_check_mark",
      "sendProgress": true,
      "dm": {
        "enabled": true,
        "policy": "open",
        "allowFrom": []
      }
    }
  }
}
```

### Config Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | bool | `false` | Enable the channel |
| `serverUrl` | str | `""` | Mattermost server base URL (e.g. `https://chat.example.com`) |
| `token` | str | `""` | Mattermost bot token (Personal Access Token) |
| `teamId` | str | `""` | Optional team ID to scope the bot to a specific team. If empty, the bot operates across all teams the bot user belongs to. |
| `allowFromMatchMode` | `"id"` / `"username"` / `"email"` | `"id"` | How to match `allowFrom` entries |
| `allowFrom` | str[] | `[]` | Allowed sender identifiers (interpreted per `allowFromMatchMode`) |
| `groupPolicy` | `"mention"` / `"open"` / `"allowlist"` | `"mention"` | Group/channel response policy |
| `groupAllowFrom` | str[] | `[]` | Channel IDs allowed when policy is `allowlist` |
| `replyInThread` | bool | `true` | Reply in thread for group messages |
| `includeThreadContext` | bool | `true` | Fetch thread history when pulled into existing thread |
| `threadContextLimit` | int | `20` | Max thread messages to fetch as context |
| `streaming` | bool | `true` | Enable token-by-token streaming |
| `streamingMaxChars` | int | `16000` | Max characters for accumulated streaming text. When exceeded, the current post is finalized and a new one is started. Keeps posts under Mattermost's 16,383 char limit with margin. |
| `reactEmoji` | str | `"eyes"` | Emoji to add on message receipt |
| `doneEmoji` | str | `"white_check_mark"` | Emoji to add after response completes |
| `sendProgress` | bool | `true` | Show "processing" status |
| `sendToolHints` | bool | `false` | Show tool call breadcrumbs as low-emphasis system posts |
| `dm.enabled` | bool | `true` | Allow DMs |
| `dm.policy` | `"open"` / `"allowlist"` | `"open"` | DM access policy |
| `dm.allowFrom` | str[] | `[]` | Allowed DM senders (when policy is `allowlist`). Uses the same `allowFromMatchMode` as the top-level `allowFrom` for matching. |

### Config Schema (Python)

The Pydantic config model lives in `nanobot/channels/mattermost.py` (same file as the channel), following the Slack pattern. It is **not** declared in `nanobot/config/schema.py` — `ChannelsConfig` uses `extra="allow"` so per-channel sections are parsed by each channel's own model:

```python
from pydantic import Field
from nanobot.config_base import Base


class MattermostDMConfig(Base):
    """Mattermost DM policy configuration."""
    enabled: bool = True
    policy: str = "open"
    allow_from: list[str] = Field(default_factory=list)


class MattermostConfig(Base):
    """Mattermost channel configuration."""
    enabled: bool = False
    server_url: str = ""
    token: str = ""
    team_id: str = ""
    allow_from_match_mode: str = "id"
    allow_from: list[str] = Field(default_factory=list)
    group_policy: str = "mention"
    group_allow_from: list[str] = Field(default_factory=list)
    reply_in_thread: bool = True
    include_thread_context: bool = True
    thread_context_limit: int = 20
    streaming: bool = True
    streaming_max_chars: int = 16000
    react_emoji: str = "eyes"
    done_emoji: str = "white_check_mark"
    send_progress: bool = True
    send_tool_hints: bool = False
    dm: MattermostDMConfig = Field(default_factory=MattermostDMConfig)
```

The channel class follows the standard discovery pattern — no changes to `nanobot/channels/__init__.py` or `nanobot/channels/registry.py` are needed. The `pkgutil` scan will auto-discover `mattermost.py`, and the channel manager will import it when `enabled: true` is set in the config.

> **Note on field aliasing**: The `Base` model at `nanobot/config_base.py` uses `alias_generator=to_camel` with `populate_by_name=True`, so snake_case Python fields (e.g. `allow_from`, `server_url`) accept camelCase JSON keys (`allowFrom`, `serverUrl`) automatically. No explicit `alias=` parameters are needed in the model — the aliasing is handled by the base class.

---

## Implementation Checklist

### Phase 1 — Core (Minimal Viable Channel)

- [x] `start()` — Connect to Mattermost WebSocket, authenticate via Bearer token, enter listen loop with auto-reconnect
- [x] `stop()` — Clean disconnect, close HTTP client
- [x] `send()` — Post messages via REST API (`POST /api/v4/posts`)
- [x] Inbound message parsing — Handle `posted` WebSocket events, extract `user_id`, `channel_id`, `message`, `root_id`, `post_id`
- [x] Bot self-message filtering — Ignore own messages
- [x] DM/channel type detection — Resolve channel type from event or API
- [x] `_handle_message()` usage — Call `BaseChannel._handle_message()` (not `bus.publish_inbound()` directly) to get permission checks and pairing code support
- [x] `default_config()` classmethod — Return `MattermostConfig().model_dump(by_alias=True)` for auto-onboard support
- [x] `is_allowed()` override — Always return `True`; do channel-aware policy checks (DM/group/allowFrom) in the inbound handler before calling `_handle_message()`

### Phase 2 — Policy & Access Control

- [x] DM policy (`open` / `allowlist`)
- [x] Group policy (`mention` / `open` / `allowlist`)
- [x] Sender allowlist with match modes: `id`, `username`, `email`
- [x] Bot @mention stripping from incoming text
- [x] Identity caching (`_usernames`, `_user_emails`, `_channel_types` dicts)

### Phase 3 — Thread Support

- [x] Thread-scoped sessions (`mattermost:{chat_id}:{root_id}`)
- [x] `replyInThread` — Set `root_id` on outbound messages when enabled
- [ ] Thread context fetching — When pulled into an existing thread (message has `root_id` that is not a reply to bot), fetch thread history via `GET /api/v4/posts/{post_id}/thread` and prepend as context
- [ ] Thread context caching — Rate-limit thread fetches per `(channel_id, root_id)` to once per session

### Phase 4 — Rich Content

- [x] File uploads — Upload files via `POST /api/v4/files`, attach `file_ids` to post
- [x] Reaction on receipt — `POST /api/v4/reactions` with `reactEmoji`
- [ ] Reaction on completion — Remove `reactEmoji`, add `doneEmoji`
- [ ] Inbound file downloads — Extract file metadata from `posted` event payload (`file_ids`), download via `GET /api/v4/files/{file_id}` using bot token
- [ ] Markdown conversion — Convert nanobot Markdown to Mattermost markdown (GFM):
  - Mattermost uses GitHub Flavored Markdown natively — most standard Markdown works as-is
  - Code fences with language tags — supported natively
  - Tables (GFM tables) — supported natively
  - Bold, italic, lists, links, headers — supported natively
  - If a custom converter is needed (e.g. for edge cases), implement `_to_gfm()` following the `_to_mrkdwn()` pattern in Slack
- [ ] Long message splitting — Split messages > 16,383 chars (Mattermost post max) across multiple posts. Split at paragraph or sentence boundaries (not mid-word), following the `split_message()` utility pattern used by Slack.

### Phase 5 — Streaming & UX

- [ ] `send_delta()` — Edit a single post in-place with accumulating text using `PUT /api/v4/posts/{post_id}` (Mattermost supports post editing)
  - On first delta: create a placeholder post, store its `post_id`
  - On subsequent deltas: edit the post in place
  - On `_stream_end`: finalize with `doneEmoji`
  - Buffer keyed by `_stream_id`, not just `chat_id`
  - When accumulated text exceeds `streamingMaxChars`: finalize current post, create new post for next chunk
- [ ] `send_reasoning_delta()` / `send_reasoning_end()` — Use ephemeral messages or a second "thinking" post that gets removed on completion
- [ ] Progress messages — Use `_progress` metadata to show ephemeral "Processing..." status
- [ ] Tool hints — When `sendToolHints` is enabled, show tool call breadcrumbs as low-emphasis system posts

### Phase 6 — Interactive Components

- [ ] Action/button support — Send posts with `actions` (Mattermost interactive message buttons via `POST /api/v4/posts` with `props.attachments[].actions`)
- [ ] Interactive button handling — Listen for `action` WebSocket events (Mattermost `action` event for button clicks), parse `context.selected_option` or `context.value`, route via `_handle_message()`
- [ ] Slash command routing — (Optional) Handle registered slash commands via Mattermost `slash_commands` integration

### Phase 7 — Resilience & Edge Cases

- [ ] WebSocket auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s) — Already present in the original implementation
- [ ] Authentication failure detection — Check bot token validity on `start()`: if WebSocket closes with error code `4001` or `status: "FAIL"`, log error and don't retry
- [ ] Rate limit handling — Detect `HTTP 429`, back off with `Retry-After` header
- [ ] Channel type cache invalidation — Handle channel deletion/archival gracefully
- [ ] File upload retries — Upload each file individually, skip failures without failing the whole post
- [ ] Graceful shutdown — Cancel in-flight requests, close WebSocket cleanly

---

## API Reference

### WebSocket Connection

Connect to `wss://{server}/api/v4/websocket`. Two authentication methods:

**Method A — Header auth (simpler, preferred):**
Pass `Authorization: Bearer <token>` as an HTTP header during the WebSocket upgrade handshake. This is what the initial implementation uses and is identical to REST API auth.

**Method B — Authentication challenge (after connect):**
```json
{
  "seq": 1,
  "action": "authentication_challenge",
  "data": {
    "token": "mattermosttokengoeshere"
  }
}
```
The server responds with `{"status": "OK", "seq_reply": 1}` on success.

The implementation should prefer Method A, which is simpler and avoids the extra round-trip.

### WebSocket Event Format

Incoming events have this shape:
```json
{
  "event": "posted",
  "data": {
    "channel_name": "town-square",
    "channel_type": "O",
    "post": "{\"id\":\"...\",\"user_id\":\"...\",\"channel_id\":\"...\",\"message\":\"hello\",\"root_id\":\"\",\"create_at\":123456789}"
  },
  "broadcast": {
    "channel_id": "...",
    "team_id": "..."
  },
  "seq": 12
}
```

The `data.post` field is a JSON-encoded string that must be parsed. The `channel_type` field in `data` can be used to avoid an extra API call for channel type resolution.

Channel type codes in `channel_type`:

| Code | Type |
|------|------|
| `O` | Public channel |
| `P` | Private channel |
| `D` | Direct message (1-on-1) |
| `G` | Group message (multi-person DM) |

When the inbound post contains file attachments, the `data.post` JSON string includes a `file_ids` array:
```json
{
  "id": "...",
  "user_id": "...",
  "channel_id": "...",
  "message": "here's the screenshot",
  "file_ids": ["abcdefghijklmnopqrstuvwxyz"],
  "root_id": "",
  "create_at": 123456789
}
```

The `file_ids` values are opaque IDs. The implementation must fetch each file via `GET /api/v4/files/{file_id}` to download the binary content.

### Interactive Button Event (`action`)

When a user clicks an interactive message button, Mattermost sends an `action` WebSocket event:

```json
{
  "event": "action",
  "data": {
    "user_id": "userid1234567890abcd",
    "channel_id": "channelid1234567890abc",
    "team_id": "teamid1234567890abcde",
    "post_id": "postid1234567890abcdef",
    "trigger_id": "triggerid1234567890abcdef...",
    "type": "button",
    "data_source": "",
    "context": {
      "selected_option": "button_label_text"
    }
  },
  "broadcast": {
    "user_id": "userid1234567890abcd",
    "channel_id": "channelid1234567890abc"
  },
  "seq": 13
}
```

The `context.selected_option` contains the button label text (same value that was sent in `actions[].button` or `actions[].name`). The implementation should extract `context.selected_option`, treat it as the user's message text, and route it through `_handle_message()`.

### Post Deleted Event (`post_deleted`)

```json
{
  "event": "post_deleted",
  "data": {
    "post": "{\"id\":\"postid1234567890abcdef\",\"delete_at\":123456789}",
    "channel_id": "channelid1234567890abc"
  },
  "broadcast": {
    "channel_id": "channelid1234567890abc"
  },
  "seq": 14
}
```

Used for best-effort cleanup of streaming state — specifically, clearing the `_stream_posts` mapping (stream ID → post ID) so that any in-flight streaming for deleted posts does not attempt further edits.

### Inbound Events (Mattermost → nanobot)

| Event | Action |
|-------|--------|
| `posted` | Incoming message. Parse post, check policy, forward to `_handle_message()`. |
| `action` | Interactive button/menu click. Parse `user_id`, `channel_id`, `context`. |
| `post_edited` | **Ignored** — editing a message does not re-trigger integrations. |
| `post_deleted` | (Best-effort) Clean up streaming post ID mapping (`_stream_posts`). |

### Outbound Calls (nanobot → Mattermost)

| Operation | API Endpoint | Method | Used By |
|-----------|-------------|--------|---------|
| Auth test | `GET /api/v4/users/me` | `start()` | Identify bot user (returns `id`, `username`, `email`) |
| Send post | `POST /api/v4/posts` | `send()` | Regular messages |
| Edit post | `PUT /api/v4/posts/{post_id}` | `send_delta()` | Streaming updates |
| Send ephemeral | `POST /api/v4/posts/ephemeral` | `send_reasoning_delta()` | Per-user messages (reasoning, progress) |
| Upload file | `POST /api/v4/files` | `send()` (media) | Multipart form: `files` + `channel_id` |
| Download file | `GET /api/v4/files/{file_id}` | Inbound file handling | Bot-authed download |
| Add reaction | `POST /api/v4/reactions` | Receipt + completion | Body: `{user_id, post_id, emoji_name}` |
| Remove reaction | `DELETE /api/v4/users/{user_id}/posts/{post_id}/reactions/{emoji_name}` | On `_stream_end` | Path-param-based; no body |
| Get user | `GET /api/v4/users/{user_id}` | Identity resolution | Returns `{id, username, email, ...}` |
| Get channel | `GET /api/v4/channels/{channel_id}` | Channel type resolution | Returns `{id, type, name, ...}` |
| Get thread | `GET /api/v4/posts/{post_id}/thread` | Thread context | Returns `{posts: [...], order: [...]}` |
| WebSocket | `wss://server/api/v4/websocket` | Inbound event stream | Persistent connection |

### File Upload Details

File upload via `POST /api/v4/files` uses `multipart/form-data`:

| Field | Type | Description |
|-------|------|-------------|
| `files` | File[] | One or more files to upload |
| `channel_id` | str | Target channel ID |
| `client_ids` | str[] (optional) | Client-generated IDs for dedup |

Response:
```json
{
  "file_infos": [
    {
      "id": "fileid1234567890abcdef",
      "user_id": "botuserid1234567890abc",
      "name": "screenshot.png",
      "extension": "png",
      "size": 102400,
      "mime_type": "image/png",
      "create_at": 123456789
    }
  ]
}
```

Attach files to a post by including `file_ids: ["fileid1234567890abcdef"]` in the post body.

### Reaction API Details

**Add reaction** (`POST /api/v4/reactions`):
```json
{
  "user_id": "botuserid1234567890abc",
  "post_id": "postid1234567890abcdef",
  "emoji_name": "eyes"
}
```
Response: `201 Created` with the reaction object.

**Remove reaction** (`DELETE /api/v4/users/{user_id}/posts/{post_id}/reactions/{emoji_name}`):

No request body — the reaction is identified entirely by path parameters. Example URL:

```
DELETE /api/v4/users/botuserid1234567890abc/posts/postid1234567890abcdef/reactions/eyes
```

### Interactive Buttons (Outbound)

Mattermost interactive message buttons use `props.attachments[].actions` in the post body:

```json
{
  "channel_id": "channelid1234567890abc",
  "message": "What would you like to do?",
  "props": {
    "attachments": [
      {
        "text": "Choose an option:",
        "actions": [
          {
            "id": "btn_approve",
            "name": "Approve",
            "integration": {
              "url": "",
              "context": {
                "option": "Approve"
              }
            }
          },
          {
            "id": "btn_reject",
            "name": "Reject",
            "integration": {
              "url": "",
              "context": {
                "option": "Reject"
              }
            }
          }
        ]
      }
    ]
  }
}
```

Key points:
- `actions[].id` — Unique action identifier (used in the `action` event's `context` as `selected_option` or via `data_source`)
- `actions[].name` — Button label text displayed to the user
- `actions[].integration.context.option` — The value returned in the `action` event. **Note**: the outbound key is `option` but the inbound event field is `selected_option` — Mattermost maps the first value from the `context` dict regardless of key name
- `actions[].integration.url` — Can be empty string for simple buttons; when filled, Mattermost sends a POST to that URL instead
- The `id` and `name` fields in the `action` event response correspond to the button clicked
- Mattermost does NOT support Slack-style Block Kit; all interactive content goes through `props.attachments`

### Ephemeral Messages

Ephemeral posts are visible only to the target user and disappear on page reload. Send via `POST /api/v4/posts/ephemeral`:

```json
{
  "user_id": "userid1234567890abcd",
  "post": {
    "channel_id": "channelid1234567890abc",
    "message": "Thinking..."
  }
}
```

The `user_id` field specifies who can see it. The `post` object uses the same shape as a regular post. Use ephemeral messages for:
- Progress indicators ("Processing your request...")
- Reasoning traces (hidden from other channel members)
- Error messages directed at a single user

**Chunking**: Ephemeral messages are subject to the same 16,383 character limit as regular posts. Long reasoning traces should be split into ~16,000 char chunks and sent as separate ephemeral messages.

**Emoji name constraint**: Mattermost emoji names must be lowercase alphanumeric plus underscores, with no colons. The `reactEmoji` and `doneEmoji` config values must conform to this pattern (e.g. `"eyes"`, `"white_check_mark"`).

**File IDs presence**: When the inbound `posted` event has no attachments, the `file_ids` field is absent from the parsed post JSON (not an empty list). The implementation must use `.get("file_ids", [])` or equivalent when parsing `data.post`.

---

## Key Design Decisions

### 1. Use `_handle_message()` Instead of Raw `bus.publish_inbound()`

The original implementation called `bus.publish_inbound()` directly, bypassing the base class's permission checks and pairing code flow. Every inbound event must route through `_handle_message()`, passing `is_dm=True` for direct messages so pairing codes work for unknown senders.

### 2. Override `is_allowed()` to Return `True`

Like Slack, Mattermost needs channel-aware policy checks (DM policy, group policy, allowFrom matching) that depend on the channel type and sender context. The `BaseChannel.is_allowed()` method only checks a flat allowlist — it cannot distinguish DMs from channels. The implementation overrides `is_allowed()` to always return `True` and performs its own policy checks before calling `_handle_message()`. This mirrors the Slack pattern at `nanobot/channels/slack.py:673`.

### 3. Thread Sessions Use the Same Pattern as Slack

Session key format: `mattermost:{channel_id}:{root_id}` (only when `root_id` is present). This mirrors Slack's `slack:{chat_id}:{thread_ts}` pattern and ensures thread-isolated conversation context.

### 4. Thread Context Fetching — Once Per Session

When the bot is mentioned in an existing thread (inbound post has a `root_id` that was not created by the bot), thread history is fetched via `GET /api/v4/posts/{root_id}/thread` and prepended as context. To avoid redundant API calls:

- A `_thread_context_attempted` set (keyed by `(channel_id, root_id)`) tracks which threads have already been queried.
- Thread context is fetched **at most once per session** — the first time a message with a given `root_id` is processed.
- If the fetch fails (network error, permissions), a warning is logged and processing continues without context.
- The `includeThreadContext` config flag disables this entirely when `false`.

### 5. No Optional Dependencies

`httpx` and `websockets` are already transitive dependencies of nanobot via other channels. No new `pyproject.toml` extras needed.

### 6. Streaming via Post Editing

Mattermost supports editing posts after creation (`PUT /api/v4/posts/{post_id}`). The streaming implementation:
- Creates the first post (delta 1) with initial text
- Edits that same post on each subsequent delta
- Adds `doneEmoji` on `_stream_end`

This provides a smooth typewriter effect similar to Slack/Telegram streaming.

### 7. Identity Resolution

Mattermost user IDs are 26-character strings. The `allowFromMatchMode` controls how `allowFrom` entries are compared:
- `"id"` — Direct string comparison with `user_id`
- `"username"` — Resolve `user_id` → username via API, compare
- `"email"` — Resolve `user_id` → email (lowercased), compare

Results are cached in `_usernames` and `_user_emails` dicts to minimize API calls.

### 8. Self-Identification on Startup

On `start()`, the implementation calls `GET /api/v4/users/me` to fetch the bot's own `id`, `username`, and `email`. These are stored as `self._self_id`, `self._self_username`, and `self._self_email` for:
- **Self-message filtering** — Ignore `posted` events where `user_id == self._self_id`
- **@mention stripping** — Remove `@{self._self_username}` from incoming message text
- **Reaction API** — Provide `user_id` when adding/removing reactions

### 9. `serverUrl` Normalization

Strip trailing `/` from `serverUrl` on initialization:
```python
self._server_url = config.serverUrl.rstrip("/")
```
This avoids double-slash bugs in URL construction like `https://server/api/v4//posts`.

Key URLs derived from the normalized base:
- REST API: `{server_url}/api/v4/{endpoint}`
- WebSocket: `wss://{ws_host}/api/v4/websocket`

The WebSocket URL extracts the host from `server_url` and replaces `https://` with `wss://` (or `http://` → `ws://`).

### 10. Streaming Sends Full Accumulated Text

Mattermost's `PUT /api/v4/posts/{post_id}` **replaces** the entire post content — it does not append. Each streaming delta must send the complete accumulated text so far. This is the same pattern Slack uses with `chat.update`.

The `send_delta()` signature matches `BaseChannel.send_delta(chat_id, delta, metadata)`. The stream ID is passed inside `metadata` as `metadata["_stream_id"]` — there is no separate `stream_id` parameter.

```python
async def send_delta(self, chat_id, delta, metadata):
    stream_id = metadata.get("_stream_id", chat_id)
    if stream_id not in self._stream_posts:
        # First delta — create post
        post = await self._create_post(chat_id, delta, root_id=...)
        self._stream_posts[stream_id] = post["id"]
    else:
        # Subsequent deltas — replace entire content
        post_id = self._stream_posts[stream_id]
        await self._edit_post(post_id, delta)  # delta is full text, not diff
```

### 11. No Deduplication Needed

Unlike Slack (which fires both `message` and `app_mention` for @-mentions), Mattermost fires a single `posted` event per message. No dedup logic is required.

### 12. Team Scoping

Mattermost supports multi-team deployments. When `teamId` is set, the implementation filters incoming events to only process messages from that team. The `broadcast.team_id` field in WebSocket events is used for this filtering. When `teamId` is empty, the bot operates across all teams the bot user belongs to.

**DM exception**: DM events (`channel_type: "D"`) may not carry a `team_id` in the broadcast field. DMs are always allowed regardless of `teamId` filtering — a user who DMs the bot should get a response even if the DM event lacks team context.

### 13. Streaming Truncation Strategy

Mattermost posts have a hard limit of 16,383 characters. During streaming, the accumulated text may exceed this limit for long responses. The implementation handles this via `streamingMaxChars` (default 16,000, with 383 chars of margin):

- When accumulated text reaches `streamingMaxChars`, the current post is finalized (add `doneEmoji`).
- A new post is created for the next chunk of streaming text.
- The `_stream_posts` mapping stores the current post ID per `stream_id`, and a new post ID replaces it when a chunk boundary is hit.

This avoids truncation errors from the Mattermost API while keeping the typewriter effect continuous.

### 14. Identity Cache Rate Limiting

When `allowFromMatchMode` is `"username"` or `"email"`, each inbound message from an uncached user triggers `GET /api/v4/users/{user_id}`. To avoid rate limit pressure:

- Results are cached in `_usernames` and `_user_emails` dicts (keyed by `user_id`).
- Cache entries are permanent for the lifetime of the channel process (usernames/emails rarely change).
- If a `GET /api/v4/users/{user_id}` call returns 429, the implementation backs off per `Retry-After` and retries (up to 3 attempts). If all retries fail, the user is treated as **not allowed** for the current message and a warning is logged. The message is dropped rather than processed without identity checks.

### 15. Interactive Button Context Mapping

Mattermost's outbound button definition and inbound click event use **different** context field names:

- **Outbound**: `props.attachments[].actions[].integration.context.option` — the developer-defined value.
- **Inbound**: `data.context.selected_option` — the value returned when the button is clicked.

The implementation maps `context.selected_option` from the inbound `action` event to the user's message text and routes it through `_handle_message()`. The `option` key name in the outbound context is arbitrary — the implementation should use a consistent key (e.g. `"option"`) and document it.

### 16. Ephemeral Message Chunking

Ephemeral messages (used for reasoning traces and progress indicators) are subject to the same 16,383 character limit as regular posts. For long reasoning traces:

- The implementation splits reasoning text into chunks of ~16,000 chars.
- Each chunk is sent as a separate ephemeral message.
- Ephemeral messages are **self-cleaning** — they disappear on page reload. On Mattermost v6.x+, the implementation may optionally call `POST /api/v4/posts/{post_id}/actions/delete_ephemeral` for explicit cleanup on `send_reasoning_end()`, but this is not required — the reload-based cleanup is sufficient.

---

## Comparison: Mattermost vs Slack

| Feature | Slack | Mattermost (Target) |
|---------|-------|-------------------|
| Inbound transport | Socket Mode (WSS) | WebSocket API |
| Outbound API | `chat.postMessage` | `POST /api/v4/posts` |
| Thread replies | `thread_ts` param | `root_id` field |
| File upload | `files_upload_v2` | `POST /api/v4/files` + `file_ids` |
| Reactions | `reactions_add` / `reactions_remove` | `POST /api/v4/reactions` |
| Streaming | None (chat.update on message) | Post editing (`PUT /api/v4/posts`) |
| Thread context | `conversations.replies` | `GET /api/v4/posts/{id}/thread` |
| Interactive buttons | Block Kit `actions` block | `props.attachments[].actions` |
| Max message length | ~40,000 chars | 16,383 chars |
| Markdown | `mrkdwn` (custom) | GFM (standard) |
| Identity | User IDs (`U...`) | User IDs (26-char hex) |
| DM channel prefix | `D` prefix | `D` channel type |
| Channel name resolution | `conversations.list` | `GET /api/v4/channels/{id}` |

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Invalid token | Log error, don't start channel |
| WebSocket disconnect | Auto-reconnect with exponential backoff (1s–30s) |
| HTTP 429 (rate limit) | Parse `Retry-After` header (seconds), wait + retry up to 3 times. Mattermost also sets `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers for visibility |
| HTTP 5xx | Raise exception (manager retries `send()`) |
| File not found (upload) | Skip file, log warning, proceed with post |
| Reaction failure | Best-effort, log debug warning |
| Thread context fetch failure | Log warning, continue without context |

---

## Files to Touch

| File | Change |
|------|--------|
| `nanobot/channels/mattermost.py` | Full channel implementation (~700–800 lines). Includes `MattermostConfig` and `MattermostDMConfig` Pydantic models. Auto-discovered by `pkgutil` scan — no changes to `__init__.py` or `registry.py` needed. |
| `nanobot/config/schema.py` | **No changes needed** — `ChannelsConfig` uses `extra="allow"`, so the `mattermost` key is accepted as a dict and parsed by `MattermostConfig` inside the channel class. |
| `docs/chat-apps.md` | Add Mattermost section with setup instructions (see below) |
| `README.md` | Add Mattermost to supported channels table |
| `tests/channels/test_mattermost_channel.py` | Tests (see Test Plan below) |

---

## Setup Docs (for `docs/chat-apps.md`)

```markdown
<details>
<summary><b>Mattermost</b></summary>

Uses **Mattermost WebSocket + REST API** — no public URL required.

**1. Create a bot token**
- In Mattermost, create or choose a bot account
- Generate a bot token with permission to read and post
- Copy the server URL (e.g. `https://your-mattermost.example.com`)
- Add the bot to the channels you want it to use

**2. Configure**

```json
{
  "channels": {
    "mattermost": {
      "enabled": true,
      "serverUrl": "https://your-mattermost.example.com",
      "token": "YOUR_MATTERMOST_BOT_TOKEN",
      "allowFromMatchMode": "id",
      "allowFrom": ["YOUR_USER_ID"],
      "groupPolicy": "mention",
      "streaming": true
    }
  }
}
```

**3. Run**

```bash
nanobot gateway
```

DM the bot directly or mention `@botname` in a channel.

> [!TIP]
> - `allowFromMatchMode`: `"id"` (default — Mattermost user IDs), `"username"`, or `"email"`.
> - `groupPolicy`: `"mention"` (default — respond only when @mentioned), `"open"` (all messages), or `"allowlist"` (specific channels via `groupAllowFrom`).
> - `streaming`: `true` (default) enables token-by-token typewriter effect. Set `false` to send complete messages at once.
> - `reactEmoji`: Emoji shown when a message is received. `doneEmoji`: Emoji shown when the response is complete.
> - File attachments are uploaded first, then linked to the outgoing post.

</details>
```

---

## Test Plan

### Unit Tests (`tests/channels/test_mattermost_channel.py`)

| Category | Test Cases |
|----------|------------|
| **Config parsing** | Valid config, missing required fields, defaults applied, camelCase alias mapping, `teamId` optional |
| **Self-identification** | `start()` calls `GET /api/v4/users/me`, stores `_self_id`, `_self_username`, `_self_email` |
| **Inbound routing** | `posted` event → parse post JSON, extract fields, route to `_handle_message()` |
| **Self-message filtering** | Own `user_id` → message ignored |
| **DM detection** | `channel_type: "D"` → `is_dm=True` passed to `_handle_message()` |
| **Channel detection** | `channel_type: "O"` / `"P"` / `"G"` → `is_dm=False` |
| **@mention stripping** | `@botname hello` → `hello` |
| **DM policy: open** | Any sender allowed |
| **DM policy: allowlist** | Only `allowFrom` senders allowed |
| **Group policy: mention** | Only messages with `@botname` are processed |
| **Group policy: open** | All messages processed |
| **Group policy: allowlist** | Only `groupAllowFrom` channel IDs processed |
| **Match mode: id** | Direct `user_id` comparison |
| **Match mode: username** | API call to resolve username, cache hit on second call |
| **Match mode: email** | API call to resolve email, cache hit on second call |
| **Identity cache** | Second message from same user does not trigger API call |
| **send()** | Calls `POST /api/v4/posts` with correct body |
| **send() with files** | Uploads files via `POST /api/v4/files`, attaches `file_ids` to post |
| **send() with thread** | Sets `root_id` on post when `replyInThread=true` |
| **Thread session key** | `mattermost:{channel_id}:{root_id}` when `root_id` present |
| **Streaming: first delta** | Creates post, stores post ID |
| **Streaming: subsequent delta** | Edits existing post with full accumulated text |
| **Streaming: chunk boundary** | Finalizes current post, creates new one at `streamingMaxChars` |
| **Streaming: stream end** | Adds `doneEmoji`, clears `_stream_posts` entry |
| **Reaction: receipt** | Adds `reactEmoji` on inbound message |
| **Reaction: completion** | Removes `reactEmoji`, adds `doneEmoji` |
| **Interactive button** | `action` event → `context.selected_option` → `_handle_message()` |
| **post_deleted** | Clears `_stream_posts` mapping for deleted post |
| **Server URL normalization** | Trailing `/` stripped, double-slash avoided |
| **Team filtering** | Events from wrong team ignored when `teamId` set |
| **Auth failure** | Invalid token → log error, channel does not start |
| **DM policy: allowlist with match mode** | DM sender matched by username or email when `dm.allowFrom` used with `allowFromMatchMode` |
| **Team filtering: DM bypass** | DM event without `team_id` in broadcast is still processed when `teamId` is set |

### Integration Tests

| Scenario | Description |
|----------|-------------|
| WebSocket reconnect | Simulate disconnect, verify auto-reconnect with backoff |
| Rate limit (429) | Mock 429 response, verify `Retry-After` backoff |
| Concurrent streaming | Two streams to same channel simultaneously, verify independent post IDs |
| Large file upload | Upload file > Mattermost limit, verify error handling |
| Thread context fetch | Existing thread with `root_id`, verify history fetched and prepended |

---

## Dependency Check

| Package | Already used by nanobot? | New dependency? |
|---------|-------------------------|-----------------|
| `httpx` | Yes (Slack, email, file download) | No |
| `websockets` | Yes (Slack Socket Mode, WeCom) | No |
| `pydantic` | Yes (all config models) | No |
| `loguru` | Yes (entire project) | No |
| `re` | stdlib | No |
| `asyncio` | stdlib | No |

No new dependencies are introduced.

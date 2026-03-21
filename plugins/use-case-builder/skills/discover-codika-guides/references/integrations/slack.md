# Slack Integration Guide

Slack provides workspace messaging, channel management, and collaboration capabilities.

> **Implementation Reference:** [n8n Slack Node on GitHub](https://github.com/n8n-io/n8n/tree/master/packages/nodes-base/nodes/Slack)

---

## 1. Integration Details

| Property | Value |
|----------|-------|
| **Integration UID** | `slack` |
| **Node Type** | `n8n-nodes-base.slack` |
| **Trigger Node Type** | `n8n-nodes-base.slackTrigger` |
| **Credential Type** | `slackOAuth2Api` |
| **Credential Level** | ORGCRED (organization-level) |

### Credential Pattern

```json
"credentials": {
  "slackOAuth2Api": {
    "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
    "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
  }
}
```

---

## 2. Codika-Specific Tips

### Error Handling Flags

| Scenario | Flag | Why |
|----------|------|-----|
| Read operations that may return empty | `alwaysOutputData: true` | Prevents workflow from stopping on 0 results |
| Write operations that may fail | `continueOnFail: true` | Allows graceful error handling |

### Message Timestamp Format

Slack uses a unique timestamp format: `1663233118.856619` (Unix seconds.microseconds). This is returned in the `ts` field of message operations and is required for:
- Updating messages
- Deleting messages
- Adding reactions
- Getting thread replies

---

## 3. Trigger Setup

The Slack Trigger node listens for events in your Slack workspace and starts the workflow when they occur.

### Trigger Events (8 events)

| Event | Value | Description |
|-------|-------|-------------|
| New Message Posted to Channel | `message` | A new message is posted |
| Bot / App Mention | `app_mention` | Bot or app is mentioned |
| Reaction Added | `reaction_added` | A reaction emoji is added |
| File Shared | `file_share` | A file is shared |
| File Made Public | `file_public` | A file is made public |
| New Public Channel Created | `channel_created` | A new public channel is created |
| New User | `team_join` | A new user joins the workspace |
| Any Event | `any_event` | Triggers on any Slack event |

### Webhook ID Requirement

The `webhookId` must be at the **node level** (sibling to `parameters`, not inside it):

```json
"webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}"
```

### Trigger Parameters

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Events | `trigger` | multiOptions | Yes | Which events to listen for (array of event values) |
| Channel | `channelId` | resourceLocator | No | Specific channel to monitor |
| Watch Workspace | `watchWorkspace` | boolean | No | Monitor all channels in workspace |
| Download Files | `downloadFiles` | boolean | No | Download attached files as binary data |

### Recommended Workflow Structure

```text
Slack Trigger → Codika Init → Business Logic → IF (success?)
                                                  ├─ Yes → Codika Submit Result
                                                  └─ No  → Codika Report Error
```

The Codika Init node must use `triggerType: "slack"`. See [third-party-triggers.md](../specific/third-party-triggers.md) for full configuration details.

### Trigger Output Structure

The Slack Trigger outputs event data. For the `message` event:

```json
{
  "type": "message",
  "channel": "C0123ABC456",
  "user": "U0123ABC456",
  "text": "Hello, world!",
  "ts": "1625000000.000100",
  "event_ts": "1625000000.000100",
  "channel_type": "channel",
  "team": "T0123ABC456"
}
```

**Key fields:**

| Field | Description |
|-------|-------------|
| `text` | Message content |
| `user` | Sender's user ID |
| `channel` | Channel ID where message was posted |
| `ts` | Message timestamp (unique message identifier) |
| `channel_type` | `channel`, `group`, or `im` |
| `team` | Workspace ID |

**Downstream access:** `$('Slack Trigger').first().json.text`, `$('Slack Trigger').first().json.user`, etc.

---

## 4. Available Resources & Operations

### Channel (16 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Archive | `archive` | Archive a channel |
| Close | `close` | Close a DM or group DM |
| Create | `create` | Create a new channel |
| Get | `get` | Get channel information |
| Get Many | `getAll` | List channels with filters |
| History | `history` | Get message history |
| Invite | `invite` | Invite users to channel |
| Join | `join` | Join a channel |
| Kick | `kick` | Remove user from channel |
| Leave | `leave` | Leave a channel |
| Member | `member` | List channel members |
| Open | `open` | Open/resume a DM |
| Rename | `rename` | Rename a channel |
| Replies | `replies` | Get thread replies |
| Set Purpose | `setPurpose` | Set channel purpose |
| Set Topic | `setTopic` | Set channel topic |
| Unarchive | `unarchive` | Unarchive a channel |

### Message (6 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Delete | `delete` | Delete a message |
| Get Permalink | `getPermalink` | Get message permalink |
| Search | `search` | Search messages |
| Send | `post` | Send a message |
| Send and Wait | `sendAndWait` | Send and wait for response |
| Update | `update` | Update a message |

### Reaction (3 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Add | `add` | Add reaction to message |
| Get | `get` | Get reactions on message |
| Remove | `remove` | Remove reaction |

### File (3 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Get | `get` | Get a file by ID |
| Get Many | `getAll` | List files with filters |
| Upload | `upload` | Upload a file |

### User (5 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Get | `info` | Get user information |
| Get Many | `getAll` | List users |
| Get Profile | `getProfile` | Get user's profile |
| Get Status | `getPresence` | Get online status |
| Update Profile | `updateProfile` | Update user's profile |

### User Group (5 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Create | `create` | Create user group |
| Disable | `disable` | Disable user group |
| Enable | `enable` | Enable user group |
| Get Many | `getAll` | List user groups |
| Update | `update` | Update user group |

### Star (3 operations)

| Operation | Value | Description |
|-----------|-------|-------------|
| Add | `add` | Star a message or file |
| Delete | `delete` | Remove star |
| Get Many | `getAll` | List starred items |

---

## 5. Parameter Reference (Complete)

This section documents **all** parameters available in the n8n Slack node.

### Common Patterns

#### Channel Selection (resourceLocator)

```json
"channelId": {
  "mode": "id",
  "value": "C0122KQ70S7E"
}
```

**Available modes:**
- `list` - Select from dropdown
- `id` - Channel ID (e.g., `C0122KQ70S7E`)
- `name` - Channel name (e.g., `#general`)
- `url` - Slack URL (e.g., `https://app.slack.com/client/TS9594PZK/C0122KQ70S7E`)

#### User Selection (resourceLocator)

```json
"user": {
  "mode": "id",
  "value": "U123AB45JGM"
}
```

**Available modes:**
- `list` - Select from dropdown
- `id` - User ID (e.g., `U123AB45JGM`)
- `username` - Username (e.g., `@john`)

---

### Message Resource Parameters

#### message:post (Send)

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Send To | `select` | options | Yes | - | `channel` or `user` |
| Channel | `channelId` | resourceLocator | If channel | - | Target channel |
| User | `user` | resourceLocator | If user | - | Target user for DM |
| Message Type | `messageType` | options | Yes | `text` | `text`, `block`, or `attachment` |
| Message Text | `text` | string | If text | - | Message content (markdown supported) |
| Blocks | `blocksUi` | string | If block | - | Block Kit JSON |
| Notification Text | `text` | string | If block | - | Fallback text for notifications |

**Options (otherOptions):**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Include Link to Workflow | `includeLinkToWorkflow` | boolean | `true` | Append workflow link |
| Custom Bot Profile | `botProfile.imageValues` | fixedCollection | - | Set bot avatar (emoji or URL) |
| Profile Photo Type | `botProfile.imageValues.profilePhotoType` | options | - | `image` or `emoji` |
| Emoji Code | `botProfile.imageValues.icon_emoji` | string | - | Emoji code (e.g., `:rocket:`) |
| Image URL | `botProfile.imageValues.icon_url` | string | - | Avatar image URL |
| Link Names | `link_names` | boolean | `false` | Make @mentions and #channels clickable |
| Reply to Message | `thread_ts.replyValues.thread_ts` | number | - | Parent message timestamp |
| Also Send to Channel | `thread_ts.replyValues.reply_broadcast` | boolean | `false` | Show reply in channel |
| Use Markdown | `mrkdwn` | boolean | `true` | Enable Slack markdown |
| Unfurl Links | `unfurl_links` | boolean | `false` | Preview text-based links |
| Unfurl Media | `unfurl_media` | boolean | `true` | Preview media links |
| Ephemeral User | `ephemeral.ephemeralValues.user` | resourceLocator | - | User to show ephemeral message |
| Send as Ephemeral | `ephemeral.ephemeralValues.ephemeral` | boolean | `true` | Make message temporary |
| Send as User | `sendAsUser` | string | - | Username to send as (requires scope) |

**Attachments (legacy - use Blocks instead):**

| Field | Property | Type | Description |
|-------|----------|------|-------------|
| Fallback Text | `fallback` | string | Required plain-text summary |
| Text | `text` | string | Attachment body |
| Title | `title` | string | Attachment title |
| Title Link | `title_link` | string | URL for title |
| Color | `color` | color | Left border color (e.g., `#ff0000`) |
| Pretext | `pretext` | string | Text before attachment |
| Author Name | `author_name` | string | Author display name |
| Author Link | `author_link` | string | Author URL |
| Author Icon | `author_icon` | string | Author avatar URL |
| Image URL | `image_url` | string | Large image |
| Thumbnail URL | `thumb_url` | string | Small thumbnail |
| Footer | `footer` | string | Footer text |
| Footer Icon | `footer_icon` | string | Footer icon URL |
| Timestamp | `ts` | number | Message timestamp |
| Fields | `fields.item` | array | Key-value fields |

#### message:update

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Channel | `channelId` | resourceLocator | Yes | - | Channel containing message |
| Message Timestamp | `ts` | number | Yes | - | Timestamp of message to update |
| Message Type | `messageType` | options | Yes | `text` | `text`, `block`, or `attachment` |
| Message Text | `text` | string | If text | - | New message content |
| Blocks | `blocksUi` | string | If block | - | New Block Kit JSON |

**Update Fields:**

| Field | Property | Type | Default | Description |
|-------|----------|------|---------|-------------|
| Link Names | `link_names` | boolean | `false` | Find and link @mentions |
| Parse | `parse` | options | `client` | `client`, `full`, or `none` |

#### message:delete

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Delete From | `select` | options | Yes | `channel` or `user` |
| Channel | `channelId` | resourceLocator | If channel | Channel containing message |
| User | `user` | resourceLocator | If user | DM with user |
| Message Timestamp | `timestamp` | number | Yes | Timestamp of message to delete |

#### message:getPermalink

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel containing message |
| Message Timestamp | `timestamp` | number | Yes | Timestamp of message |

#### message:search

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Search Query | `query` | string | Yes | - | Text to search for |
| Sort By | `sort` | options | No | `desc` | `desc`, `asc`, or `relevance` |
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 25 | Max results (1-50) |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Search in Channel | `searchChannel` | multiOptions | Filter by channel IDs |

---

### Channel Resource Parameters

#### channel:create

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Channel | `channelId` | string | Yes | - | New channel name |
| Visibility | `channelVisibility` | options | Yes | `public` | `public` or `private` |

#### channel:get

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel to get info for |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Include Num Members | `includeNumMembers` | boolean | `false` | Include member count |

#### channel:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

**Filters:**

| Filter | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Exclude Archived | `excludeArchived` | boolean | `false` | Hide archived channels |
| Types | `types` | multiOptions | `["public_channel"]` | `public_channel`, `private_channel`, `mpim`, `im` |

#### channel:history

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Channel | `channelId` | resourceLocator | Yes | - | Channel to get history from |
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

**Filters:**

| Filter | Property | Type | Description |
|--------|----------|------|-------------|
| Inclusive | `inclusive` | boolean | Include boundary timestamps |
| Latest | `latest` | dateTime | End of time range |
| Oldest | `oldest` | dateTime | Start of time range |

#### channel:invite

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel to invite to |
| User IDs | `userIds` | multiOptions | Yes | Users to invite |

#### channel:kick

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel to remove from |
| User ID | `userId` | options | Yes | User to remove |

#### channel:member

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Channel | `channelId` | resourceLocator | Yes | - | Channel to list members |
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 100 | Max results |
| Resolve Data | `resolveData` | boolean | No | `false` | Get full user objects |

#### channel:open

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Channel ID | `channelId` | string | Resume existing DM by ID |
| Return IM | `returnIm` | boolean | Get full IM definition |
| Users | `users` | multiOptions | Users to start DM with |

#### channel:rename

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel to rename |
| Name | `name` | string | Yes | New channel name |

#### channel:replies

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Channel | `channelId` | resourceLocator | Yes | - | Channel containing thread |
| Message Timestamp | `ts` | number | Yes | - | Parent message timestamp |
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

**Filters:**

| Filter | Property | Type | Description |
|--------|----------|------|-------------|
| Inclusive | `inclusive` | boolean | Include boundary timestamps |
| Latest | `latest` | string | End of time range |
| Oldest | `oldest` | string | Start of time range |

#### channel:setPurpose / channel:setTopic

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel to update |
| Purpose/Topic | `purpose`/`topic` | string | Yes | New text |

#### channel:archive / channel:unarchive / channel:join / channel:leave / channel:close

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Target channel |

---

### Reaction Resource Parameters

#### reaction:add / reaction:remove

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel containing message |
| Message Timestamp | `timestamp` | number | Yes | Message to react to |
| Emoji Code | `name` | string | Yes | Emoji code (e.g., `+1`, `:rocket:`) |

#### reaction:get

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Channel | `channelId` | resourceLocator | Yes | Channel containing message |
| Message Timestamp | `timestamp` | number | Yes | Message to get reactions |

---

### File Resource Parameters

#### file:get

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| File ID | `fileId` | string | Yes | ID of file to retrieve |

#### file:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

**Filters:**

| Filter | Property | Type | Description |
|--------|----------|------|-------------|
| Channel | `channelId` | options | Filter by channel |
| Show Hidden | `showFilesHidden` | boolean | Show truncated old files |
| Timestamp From | `tsFrom` | string | Filter files created after |
| Timestamp To | `tsTo` | string | Filter files created before |
| Types | `types` | multiOptions | `all`, `gdocs`, `images`, `pdfs`, `snippets`, `spaces`, `zips` |
| User | `userId` | options | Filter by creator |

#### file:upload

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| File Property | `binaryPropertyName` | string | Yes | Binary property name (default: `data`) |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Channel | `channelId` | options | Channel to upload to |
| File Name | `fileName` | string | Override file name |
| Initial Comment | `initialComment` | string | Message with file |
| Thread Timestamp | `threadTs` | string | Upload as thread reply |
| Title | `title` | string | File title |

---

### User Resource Parameters

#### user:info / user:getProfile

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| User | `user` | resourceLocator | Yes | User to get info for |

#### user:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

#### user:getPresence

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| User | `user` | resourceLocator | Yes | User to get status |

#### user:updateProfile

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Custom Fields | `customFieldUi.customFieldValues` | array | Set custom profile fields |
| Email | `email` | string | User email (admin only) |
| First Name | `first_name` | string | First name |
| Last Name | `last_name` | string | Last name |
| Status Emoji | `status.set_status.status_emoji` | string | Status emoji (e.g., `:mountain_railway:`) |
| Status Expiration | `status.set_status.status_expiration` | dateTime | When status expires |
| Status Text | `status.set_status.status_text` | string | Status message (max 100 chars) |
| User ID | `user` | string | Target user (admin only) |

---

### User Group Resource Parameters

#### userGroup:create

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Name | `name` | string | Yes | Unique group name |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Channels | `channelIds` | multiOptions | - | Default channels |
| Description | `description` | string | - | Short description |
| Handle | `handle` | string | - | @mention handle |
| Include Count | `include_count` | boolean | `true` | Include user count |

#### userGroup:disable / userGroup:enable

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| User Group ID | `userGroupId` | string | Yes | Group to enable/disable |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Include Count | `include_count` | boolean | `true` | Include user count |

#### userGroup:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 100 | Max results (1-500) |

**Options:**

| Option | Property | Type | Default | Description |
|--------|----------|------|---------|-------------|
| Include Count | `include_count` | boolean | `true` | Include user count |
| Include Disabled | `include_disabled` | boolean | `true` | Show disabled groups |
| Include Users | `include_users` | boolean | `true` | Include member list |

#### userGroup:update

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| User Group ID | `userGroupId` | string | Yes | Group to update |

**Update Fields:**

| Field | Property | Type | Description |
|-------|----------|------|-------------|
| Channels | `channels` | multiOptions | New default channels |
| Description | `description` | string | New description |
| Handle | `handle` | string | New @mention handle |
| Include Count | `include_count` | boolean | Include user count |
| Name | `name` | string | New group name |

---

### Star Resource Parameters

#### star:add

| Parameter | Property | Type | Required | Description |
|-----------|----------|------|----------|-------------|
| Target | `target` | options | Yes | `message` or `file` |
| Channel | `channelId` | resourceLocator | Yes | Channel containing item |
| File ID | `fileId` | string | If file | File to star |
| Message Timestamp | `timestamp` | number | If message | Message to star |

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| File Comment | `fileComment` | string | Comment to star |

#### star:delete

**Options:**

| Option | Property | Type | Description |
|--------|----------|------|-------------|
| Channel | `channelId` | options | Channel containing item |
| File ID | `fileId` | string | File to unstar |
| File Comment | `fileComment` | string | Comment to unstar |
| Message Timestamp | `timestamp` | number | Message to unstar |

#### star:getAll

| Parameter | Property | Type | Required | Default | Description |
|-----------|----------|------|----------|---------|-------------|
| Return All | `returnAll` | boolean | No | `false` | Return all results |
| Limit | `limit` | number | If !returnAll | 50 | Max results (1-100) |

---

## 6. Common Mistakes

| Wrong | Correct | Issue |
|-------|---------|-------|
| `operation: "send"` | `operation: "post"` | Wrong operation name |
| `channel: "C123..."` | `channelId: { mode: "id", value: "C123..." }` | Must use resourceLocator |
| `emoji: "thumbsup"` | `name: "+1"` | Reactions use `name` field |
| `userId` in message | `user` with resourceLocator | Wrong field name for DM |

---

## 7. Node Examples

### 7.0 Slack Trigger

```json
{
  "parameters": {
    "trigger": ["message"],
    "channelId": {
      "__rl": true,
      "value": "={{ $json.channelId }}",
      "mode": "id"
    }
  },
  "type": "n8n-nodes-base.slackTrigger",
  "typeVersion": 1,
  "position": [250, 300],
  "id": "slack-trigger-001",
  "name": "Slack Trigger",
  "webhookId": "{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}",
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  }
}
```

### 7.1 Send Message to Channel

```json
{
  "parameters": {
    "resource": "message",
    "operation": "post",
    "select": "channel",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "messageType": "text",
    "text": "={{ $json.messageText }}",
    "otherOptions": {}
  },
  "id": "slack-post-001",
  "name": "Send Slack Message",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  }
}
```

### 7.2 Send Message with Blocks

```json
{
  "parameters": {
    "resource": "message",
    "operation": "post",
    "select": "channel",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "messageType": "block",
    "blocksUi": "={{ JSON.stringify([{\"type\":\"section\",\"text\":{\"type\":\"mrkdwn\",\"text\":\"*New Event*\\n\" + $json.description}}]) }}",
    "text": "New Event Notification",
    "otherOptions": {}
  },
  "id": "slack-blocks-001",
  "name": "Send Block Message",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  }
}
```

### 7.3 Send Direct Message to User

```json
{
  "parameters": {
    "resource": "message",
    "operation": "post",
    "select": "user",
    "user": {
      "mode": "id",
      "value": "={{ $json.userId }}"
    },
    "messageType": "text",
    "text": "Hello! This is a direct message.",
    "otherOptions": {}
  },
  "id": "slack-dm-001",
  "name": "Send DM",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  }
}
```

### 7.4 Reply to Thread

```json
{
  "parameters": {
    "resource": "message",
    "operation": "post",
    "select": "channel",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "messageType": "text",
    "text": "This is a thread reply",
    "otherOptions": {
      "thread_ts": {
        "replyValues": {
          "thread_ts": "={{ $json.messageTs }}",
          "reply_broadcast": false
        }
      }
    }
  },
  "id": "slack-thread-001",
  "name": "Reply to Thread",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  }
}
```

### 7.5 Get Channel Info

```json
{
  "parameters": {
    "resource": "channel",
    "operation": "get",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "options": {
      "includeNumMembers": true
    }
  },
  "id": "slack-get-channel-001",
  "name": "Get Channel",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.6 Get All Channels

```json
{
  "parameters": {
    "resource": "channel",
    "operation": "getAll",
    "returnAll": false,
    "limit": 50,
    "filters": {
      "excludeArchived": true,
      "types": ["public_channel", "private_channel"]
    }
  },
  "id": "slack-getall-channels-001",
  "name": "Get All Channels",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.7 Get Channel History

```json
{
  "parameters": {
    "resource": "channel",
    "operation": "history",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "returnAll": false,
    "limit": 100,
    "filters": {
      "oldest": "={{ $now.minus({days: 7}).toISO() }}",
      "latest": "={{ $now.toISO() }}"
    }
  },
  "id": "slack-history-001",
  "name": "Get Channel History",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.8 Create Channel

```json
{
  "parameters": {
    "resource": "channel",
    "operation": "create",
    "channelId": "new-channel-name",
    "channelVisibility": "private"
  },
  "id": "slack-create-channel-001",
  "name": "Create Channel",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 7.9 Add Reaction

```json
{
  "parameters": {
    "resource": "reaction",
    "operation": "add",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "timestamp": "={{ $json.messageTs }}",
    "name": "+1"
  },
  "id": "slack-reaction-001",
  "name": "Add Reaction",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 7.10 Upload File

```json
{
  "parameters": {
    "resource": "file",
    "operation": "upload",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "binaryPropertyName": "data",
    "options": {
      "fileName": "={{ $json.fileName }}"
    }
  },
  "id": "slack-upload-001",
  "name": "Upload File",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 7.11 Get User Info

```json
{
  "parameters": {
    "resource": "user",
    "operation": "info",
    "user": {
      "mode": "id",
      "value": "={{ $json.userId }}"
    }
  },
  "id": "slack-user-info-001",
  "name": "Get User Info",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.12 Search Messages

```json
{
  "parameters": {
    "resource": "message",
    "operation": "search",
    "query": "project update",
    "sort": "desc",
    "returnAll": false,
    "limit": 25,
    "options": {
      "searchChannel": ["C0122KQ70S7E"]
    }
  },
  "id": "slack-search-001",
  "name": "Search Messages",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "alwaysOutputData": true
}
```

### 7.13 Star a Message

```json
{
  "parameters": {
    "resource": "star",
    "operation": "add",
    "target": "message",
    "channelId": {
      "mode": "id",
      "value": "={{ $json.channelId }}"
    },
    "timestamp": "={{ $json.messageTs }}",
    "options": {}
  },
  "id": "slack-star-001",
  "name": "Star Message",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

### 7.14 Create User Group

```json
{
  "parameters": {
    "resource": "userGroup",
    "operation": "create",
    "name": "engineering-team",
    "Options": {
      "handle": "engineering",
      "description": "Engineering team members",
      "include_count": true
    }
  },
  "id": "slack-usergroup-001",
  "name": "Create User Group",
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "position": [600, 0],
  "credentials": {
    "slackOAuth2Api": {
      "id": "{{ORGCRED_SLACK_ID_DERCGRO}}",
      "name": "{{ORGCRED_SLACK_NAME_DERCGRO}}"
    }
  },
  "continueOnFail": true
}
```

---

## 8. Troubleshooting

| Error | Cause | Solution |
|-------|-------|----------|
| `channel_not_found` | Invalid channel ID | Verify channel ID format and permissions |
| `not_in_channel` | Bot not in channel | Invite bot to channel first |
| `message_not_found` | Invalid timestamp | Verify message timestamp format |
| `already_reacted` | Duplicate reaction | Check before adding reaction |
| `invalid_blocks` | Malformed Block Kit JSON | Validate JSON at Block Kit Builder |
| Empty result from getAll | No matching items | Add `alwaysOutputData: true` |
| Write operation fails silently | Permission error | Add `continueOnFail: true` |

---

## 9. Settings

```json
{
  "settings": {
    "executionOrder": "v1",
    "errorWorkflow": "{{ORGSECRET_ERROR_WORKFLOW_ID_TERCESORG}}"
  }
}
```

---

## 10. Requirements Checklist

**Trigger workflows:**
- [ ] Trigger node has `webhookId` at node level (not inside parameters)
- [ ] `webhookId` value: `{{USERDATA_PROCESS_INSTANCE_UID_ATADRESU}}`
- [ ] Trigger events configured appropriately (array of event values)
- [ ] Codika Init immediately after trigger with `triggerType: "slack"`

**Action nodes:**
- [ ] Credentials use ORGCRED pattern
- [ ] Channel references use resourceLocator with `mode` property
- [ ] User references use resourceLocator with `mode` property
- [ ] Message post uses `select` parameter (`channel` or `user`)
- [ ] Reactions use `name` field (emoji code like `+1`, not emoji)
- [ ] Read operations include `alwaysOutputData: true`
- [ ] Write operations include `continueOnFail: true`
- [ ] Text fallback provided when using blocks
- [ ] Error workflow configured

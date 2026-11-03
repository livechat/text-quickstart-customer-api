## 1. Backend authorization

Configure the backend credentials:

```bash
export TEXT_CLIENT_ID='YOUR_CLIENT_ID'
export TEXT_CLIENT_SECRET='YOUR_CLIENT_SECRET'
export TEXT_CALLBACK_URI='http://localhost:8080/callback'
```

Initially, an Owner/Admin signs in and authorizes the app in a browser. Open the following URL after substituting and URL-encoding values. `redirect_uri` is the URL-encoded `TEXT_CALLBACK_URI`; it must be on the **Redirect URI whitelist** and match the value used in the code exchange exactly. Generate a random `state`; your backend callback must validate it and read the returned `code`:

```text
https://accounts.livechat.com/?response_type=code&client_id=CLIENT_ID&redirect_uri=http%3A%2F%2Flocalhost%3A8080%2Fcallback&state=RANDOM_STATE
```

Exchange the code on the backend — [authorization code exchange](https://platform.text.com/docs/authorization/oauth-authorization#step-4-exchange-code-for-access-token-and-refresh-token):

```bash
export AUTHORIZATION_CODE='CODE_RECEIVED_BY_BACKEND_CALLBACK'

curl --fail-with-body --silent --show-error \
  -X POST 'https://accounts.livechat.com/v2/token' \
  --data-urlencode 'grant_type=authorization_code' \
  --data-urlencode "client_id=$TEXT_CLIENT_ID" \
  --data-urlencode "client_secret=$TEXT_CLIENT_SECRET" \
  --data-urlencode "redirect_uri=$TEXT_CALLBACK_URI" \
  --data-urlencode "code=$AUTHORIZATION_CODE"
```

Response excerpt:

```json
{
  "access_token": "AGENT_ACCESS_TOKEN",
  "refresh_token": "AGENT_REFRESH_TOKEN",
  "organization_id": "ORGANIZATION_ID",
  "expires_in": 28800,
  "scope": "customers:own"
}
```

**Limits**: 20 valid access tokens and refresh tokens per client per account.

Copy the returned values:

```bash
export AGENT_TOKEN='COPY_ACCESS_TOKEN_FROM_RESPONSE'
export REFRESH_TOKEN='COPY_REFRESH_TOKEN_FROM_RESPONSE'
export TEXT_ORGANIZATION_ID='COPY_ORGANIZATION_ID_FROM_RESPONSE'
```

Refresh the agent token before expiry — [refresh token grant](https://platform.text.com/docs/authorization/oauth-authorization#using-the-refresh-token):

```bash
curl --fail-with-body --silent --show-error \
  -X POST 'https://accounts.livechat.com/v2/token' \
  --data-urlencode 'grant_type=refresh_token' \
  --data-urlencode "client_id=$TEXT_CLIENT_ID" \
  --data-urlencode "client_secret=$TEXT_CLIENT_SECRET" \
  --data-urlencode "refresh_token=$REFRESH_TOKEN"

export AGENT_TOKEN='COPY_NEW_ACCESS_TOKEN_FROM_REFRESH_RESPONSE'
export REFRESH_TOKEN='COPY_REFRESH_TOKEN_FROM_REFRESH_RESPONSE'
```

Store tokens securely. Reuse and refresh this authorization across runs; browser interaction is needed again only if authorization expires or is revoked. This shows standard server-side code grant; the [OAuth reference](https://platform.text.com/docs/authorization/oauth-authorization#authorization-code-grant) also documents recommended **PKCE**.

## 2. Create a new customer

Use the **agent token grant**. Omitting `entity_id` creates a **new customer identity** on each request. [New customer grant](https://platform.text.com/docs/authorization/customer-authorization#agent-token-grant)

```bash
curl --fail-with-body --silent --show-error \
  -X POST 'https://accounts.livechat.com/v2/customer/token' \
  -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"grant_type\":\"agent_token\",\"client_id\":\"$TEXT_CLIENT_ID\",\"response_type\":\"token\"}"
```

Response excerpt:

```json
{
  "access_token": "CUSTOMER_ACCESS_TOKEN",
  "entity_id": "CUSTOMER_ID",
  "expires_in": 28800,
  "organization_id": "ORGANIZATION_ID"
}
```

```bash
export CUSTOMER_TOKEN='COPY_CUSTOMER_ACCESS_TOKEN_FROM_RESPONSE'
export CUSTOMER_ID='COPY_ENTITY_ID_FROM_RESPONSE'
```

The returned organization must match `TEXT_ORGANIZATION_ID` from the agent token response. Use `CUSTOMER_TOKEN` for chat calls. Retain `CUSTOMER_ID`; to renew a token for the **same customer**, include `entity_id` — [existing customer grant](https://platform.text.com/docs/authorization/customer-authorization#agent-token-grant):

```bash
curl --fail-with-body --silent --show-error \
  -X POST 'https://accounts.livechat.com/v2/customer/token' \
  -H "Authorization: Bearer $AGENT_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"grant_type\":\"agent_token\",\"client_id\":\"$TEXT_CLIENT_ID\",\"response_type\":\"token\",\"entity_id\":\"$CUSTOMER_ID\"}"

export CUSTOMER_TOKEN='COPY_NEW_CUSTOMER_ACCESS_TOKEN_FROM_RESPONSE'
```

## 2. HTTP flow

Use the **customer token** and required `organization_id` query parameter. Replace returned IDs before continuing; this demonstration is independent of the RTM flow. [Web API methods and headers](https://platform.text.com/docs/messaging/customer-chat-api#methods)

**Start a chat** — [`start_chat`](https://platform.text.com/docs/messaging/customer-chat-api#start-chat). `continuous: true` permits starting without an online group. An available agent is needed for an immediate reply.

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/start_chat?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data '{"continuous":true}'
```

Response: `{"chat_id":"CHAT_ID","thread_id":"THREAD_ID"}`. Save the chat ID:

```bash
export CHAT_ID='COPY_CHAT_ID_FROM_RESPONSE'
```

**Send text** — [`send_event`](https://platform.text.com/docs/messaging/customer-chat-api#send-event).

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/send_event?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"chat_id\":\"$CHAT_ID\",\"event\":{\"type\":\"message\",\"text\":\"Hello, I need help with my order.\",\"recipients\":\"all\"}}"
```

Response: `{"event_id":"EVENT_ID"}`.

**Fetch the chat and its latest thread** — [`get_chat`](https://platform.text.com/docs/messaging/customer-chat-api#get-chat).

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/get_chat?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"chat_id\":\"$CHAT_ID\"}"
```

Response excerpt:

```json
{
  "id": "CHAT_ID",
  "thread": {
    "id": "THREAD_ID",
    "active": true,
    "events": [{ "id": "EVENT_ID", "type": "message", "text": "Hello, I need help with my order." }]
  }
}
```

**End the chat** — [`deactivate_chat`](https://platform.text.com/docs/messaging/customer-chat-api#deactivate-chat) closes the current thread. Its request field is **`id`**. Success: HTTP `200 OK`, no response payload.

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/deactivate_chat?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"id\":\"$CHAT_ID\"}"
```

For a returning customer, find conversations with [`list_chats`](https://platform.text.com/docs/messaging/customer-chat-api#list-chats):

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/list_chats?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data '{}'
```

Reuse an active chat. Reopen an inactive one with [`resume_chat`](https://platform.text.com/docs/messaging/customer-chat-api#resume-chat):

```bash
curl --fail-with-body --silent --show-error \
  -X POST "https://api.livechatinc.com/v3.6/customer/action/resume_chat?organization_id=$TEXT_ORGANIZATION_ID" \
  -H "Authorization: Bearer $CUSTOMER_TOKEN" -H 'Content-Type: application/json' \
  --data "{\"chat\":{\"id\":\"$CHAT_ID\"}}"
```

`list_chats` returns summaries; `get_chat` returns one thread. Use paginated [`list_threads`](https://platform.text.com/docs/messaging/customer-chat-api#list-threads) for history across threads.

## 3. Equivalent RTM / WebSocket flow

For a fresh demonstration, repeat section 3 to create another new customer; reuse the OAuth agent authorization from section 2. Connect with a WebSocket client, e.g. `wscat`, then send JSON frames. Substitute actual token/IDs in frames; shell variables are not expanded inside interactive input. [RTM endpoint](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#methods)

```bash
wscat -c "wss://api.livechatinc.com/v3.6/customer/rtm/ws?organization_id=$TEXT_ORGANIZATION_ID"
```

**OUT** means client → server; **IN** means server → client. IN examples are **excerpts**, omitting unrelated fields. Match responses by `request_id`, check `success`, and handle `type: "push"` separately; pushes can interleave with responses.

**Log in within 30 seconds** — [`login`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#login). The token string includes **`Bearer `**.

OUT:

```json
{ "request_id": "1", "action": "login", "payload": { "token": "Bearer CUSTOMER_TOKEN" } }
```

IN:

```json
{
  "request_id": "1",
  "action": "login",
  "type": "response",
  "success": true,
  "payload": { "customer": { "id": "CUSTOMER_ID" }, "has_active_thread": false, "chats": [] }
}
```

**Start** — [`start_chat`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#start-chat).

OUT:

```json
{ "request_id": "2", "action": "start_chat", "payload": { "continuous": true } }
```

IN:

```json
{
  "request_id": "2",
  "action": "start_chat",
  "type": "response",
  "success": true,
  "payload": { "chat_id": "CHAT_ID", "thread_id": "THREAD_ID" }
}
```

The [`incoming_chat`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-pushes#incoming_chat) push carries the new chat and thread (excerpt):

IN:

```json
{
  "version": "3.6",
  "action": "incoming_chat",
  "type": "push",
  "payload": {
    "requester_id": "CUSTOMER_ID",
    "chat": { "id": "CHAT_ID", "thread": { "id": "THREAD_ID", "active": true } }
  }
}
```

**Send text** — [`send_event`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#send-event).

OUT:

```json
{
  "request_id": "3",
  "action": "send_event",
  "payload": {
    "chat_id": "CHAT_ID",
    "event": { "type": "message", "text": "Hello, I need help with my order.", "recipients": "all" }
  }
}
```

IN:

```json
{
  "request_id": "3",
  "action": "send_event",
  "type": "response",
  "success": true,
  "payload": { "event_id": "EVENT_ID" }
}
```

An agent reply arrives as an [`incoming_event`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-pushes#incoming_event) push:

IN:

```json
{
  "version": "3.6",
  "action": "incoming_event",
  "type": "push",
  "payload": {
    "chat_id": "CHAT_ID",
    "thread_id": "THREAD_ID",
    "event": {
      "id": "REPLY_EVENT_ID",
      "type": "message",
      "text": "How can I help?",
      "author_id": "AGENT_ID",
      "recipients": "all"
    }
  }
}
```

The same push also reports customer messages: `author_id` identifies the sender. Deduplicate events by `event.id` when combining pushes with fetched history.

**Fetch the chat** — [`get_chat`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#get-chat).

OUT:

```json
{ "request_id": "4", "action": "get_chat", "payload": { "chat_id": "CHAT_ID" } }
```

IN:

```json
{
  "request_id": "4",
  "action": "get_chat",
  "type": "response",
  "success": true,
  "payload": {
    "id": "CHAT_ID",
    "thread": {
      "id": "THREAD_ID",
      "active": true,
      "events": [{ "id": "EVENT_ID", "type": "message", "text": "Hello, I need help with my order." }]
    }
  }
}
```

**End the chat** — [`deactivate_chat`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#deactivate-chat).

OUT:

```json
{ "request_id": "5", "action": "deactivate_chat", "payload": { "id": "CHAT_ID" } }
```

IN:

```json
{ "request_id": "5", "action": "deactivate_chat", "type": "response", "success": true, "payload": {} }
```

The [`chat_deactivated`](https://platform.text.com/docs/messaging/customer-chat-api/rtm-pushes#chat_deactivated) push confirms the active thread closed:

IN:

```json
{
  "version": "3.6",
  "action": "chat_deactivated",
  "type": "push",
  "payload": { "chat_id": "CHAT_ID", "thread_id": "THREAD_ID", "user_id": "CUSTOMER_ID" }
}
```

Throughout the authenticated session, **send ping every 15 seconds**; idle connections close after approximately 30 seconds. [Server pinging](https://platform.text.com/docs/messaging/customer-chat-api/rtm-reference#server-pinging)

OUT:

```json
{ "request_id": "ping-1", "action": "ping", "payload": {} }
```

IN:

```json
{ "request_id": "ping-1", "action": "ping", "type": "response", "success": true }
```

Closing the WebSocket only disconnects the client; use `deactivate_chat` to end the thread. Pushes require no registration and have no retries; fetch state again after reconnecting. [RTM pushes](https://platform.text.com/docs/messaging/customer-chat-api/rtm-pushes)

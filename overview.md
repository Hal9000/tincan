# Private Messaging System — Design Notes

## Purpose

Build a small, private communication system for humans, assistants, bots, and their devices.

Expected scale:

- One primary user
- Several assistants or bots
- Possibly one or two additional people

The system does not need public federation, massive scalability, advertising, public discovery, or support for millions of untrusted users.

The objective is genuine identity and direct conversation rather than routing every assistant through a generic notification provider.

Example:

```text
OSF:
What year was the last 990-N filed?

Hal:
2024

OSF:
Thank you. Continuing the job.
```

## Core Model

A central server controlled by its owner maintains logical identities for humans, assistants, and bots:

- `hal`
- `osf`
- `personal-admin`
- `writing-assistant`

An identity is not an IP address or physical device. It represents a participant that can send and receive messages.

A directory entry may contain:

- Stable identity
- Display name
- Participant type: human, assistant, or service
- Icon or avatar
- Permissions
- Associated devices or server processes
- Optional public encryption keys
- Status or availability information

## Users and Devices

A user is separate from a device.

Example:

```text
User: Hal

Devices:
- iPhone
- laptop
- iPad
```

Messages belong to Hal’s account or mailbox, not to one device. Every authorized device can synchronize and display the same history.

Each device has:

- A stable device identifier
- Authentication credentials
- Synchronization position
- Last-seen time
- Optional push-notification subscription
- Revocation state

Bots use the same conceptual protocol. A bot may run continuously on the central server rather than on a phone or laptop.

## Network Topology

Devices do not need publicly reachable addresses.

Phones and laptops often:

- Sit behind NAT
- Change networks
- Sleep
- Lose connectivity
- Receive different IP addresses

Each device therefore creates an outbound connection to the central server. The server routes and stores messages.

The directory maps logical identities to accounts and devices, not to current network addresses.

## Message Storage

The central server is the authoritative message store.

A minimal message record might contain:

- Message ID
- Mailbox sequence number
- Sender identity
- Recipient identity or conversation
- Message type
- Body or structured payload
- Creation time
- Associated interaction or job
- Client-generated idempotency key

Each mailbox has an increasing sequence number.

A reconnecting device can request:

```text
SYNC after sequence 482
```

The server returns messages 483 onward.

This makes disconnection normal. A device does not need to remain continuously connected to retain a complete history.

## Protocol Style

The natural model is message-oriented rather than resource-oriented.

Possible protocol messages:

- `AUTHENTICATE`
- `REGISTER_DEVICE`
- `SYNC`
- `MESSAGE`
- `QUESTION`
- `ANSWER`
- `ACKNOWLEDGE`
- `MARK_READ`
- `SUBSCRIBE`
- `UNSUBSCRIBE`
- `PING`
- `ERROR`

Example exchange:

```text
CLIENT -> AUTHENTICATE
CLIENT -> SYNC after 482
SERVER -> MESSAGE 483
CLIENT -> ACKNOWLEDGE 483
CLIENT -> SEND message client-7741
SERVER -> ACCEPTED client-7741 as message 484
```

A human answer might be:

```json
{
  "type": "answer",
  "client_message_id": "iphone-7741",
  "interaction_id": "int-82",
  "question_id": "q-91",
  "from": "hal",
  "to": "osf",
  "text": "2024"
}
```

The client-generated message ID prevents duplicate submission when a device retries after losing connectivity.

## Data Format

JSON is sufficient initially because it is:

- Readable
- Easy to debug
- Widely supported
- Adequate for very low traffic

A binary representation such as CBOR could be introduced later without changing the conceptual protocol.

Protocol messages should include a version so clients and servers can evolve without silently misinterpreting one another.

## Transport

Authenticated encryption is essential. HTTP semantics are not.

The system needs:

- Confidentiality
- Integrity
- Server authentication

It does not inherently need:

- `GET`
- `POST`
- `PUT`
- Resource-oriented URLs
- Protocol-colon-slash-slash notation

A custom protocol could run over:

- TLS
- QUIC
- WebSockets secured with TLS
- Another established encrypted transport

WebSockets are a practical starting point because they provide bidirectional frames, work through common firewalls and proxies, and have mature browser, phone, and server libraries.

A WebSocket begins with an HTTP-compatible handshake, but subsequent frames can carry the system’s own domain messages. HTTP is therefore an adaptation and deployment convenience, not the conceptual application protocol.

Native clients could use raw TLS or QUIC, but that adds networking and deployment work without an immediate user benefit.

## Foreground Delivery

While a phone or desktop client is open and connected:

```text
Server -> WebSocket -> Client
```

The message can appear immediately.

Desktop applications can usually keep a connection open for long periods and reconnect when the network changes.

## iPhone Background Delivery

An iPhone application cannot depend on an always-open WebSocket while it is in the background.

iOS normally suspends background applications. The socket stops operating, and the operating system may terminate the process entirely. A suspended application cannot decide to reconnect frequently.

The proper background flow is:

```text
Server -> Apple push infrastructure
Apple -> wakes or notifies the iPhone
iPhone client -> reconnects to the central server
iPhone client -> synchronizes new messages
```

Apple’s push system is only a wake-up and notification mechanism. It does not need to become the messaging system.

The push payload can contain an opaque event or mailbox identifier. The actual message remains on the private server and is retrieved through the custom protocol.

If push infrastructure is omitted, messages may not appear until the user manually opens the application.

## PWA Versus Native Client

A Progressive Web App could provide the first phone and desktop client.

Advantages:

- One codebase
- Installable on phone and desktop
- Access to Web Push
- No initial native application
- Uses the existing web application foundation

On iPhone, the user must add the application to the Home Screen and grant notification permission.

A native iPhone application could be built later for deeper platform integration and direct APNs control. That introduces application signing, Apple developer configuration, distribution, updates, and more platform-specific code.

## Notification Presentation

The operating system will identify the receiving application, just as it identifies Messages or Signal.

Within that application, the notification can identify the actual sender:

```text
OSF

What year was the last 990-N filed?
```

The assistant identity, display name, and icon come from the private directory. They are not constrained by the identity model of an unrelated notification provider.

Tapping the notification opens the relevant conversation or interaction.

## Replies

Replies are ordinary protocol messages rather than web-form submissions.

The client displays the conversation, accepts the answer, assigns an idempotency key, and sends it to the server.

The server:

1. Authenticates the sending device.
2. Verifies that the user may answer the question.
3. Rejects duplicates safely.
4. Persists the answer.
5. Marks the question answered.
6. Makes the blocked job runnable when all required answers exist.
7. Synchronizes the result to the user’s other devices.
8. Later resumes the assistant through normal job execution.

## Authentication and Device Enrollment

The system needs a controlled way to add devices.

A possible initial process:

1. The user signs into the server.
2. The server issues a short-lived enrollment code or QR code.
3. The new device presents the code.
4. The server creates a device record and long-lived device credential.
5. The credential is stored in the device’s secure storage.
6. Lost devices can be revoked centrally.

The protocol should distinguish:

- Human identity
- Assistant identity
- Device identity
- Session credential

Passwords do not need to be transmitted repeatedly after enrollment.

## Security Model

For the first private implementation, the central server can be trusted with message contents.

TLS protects communication between devices and the server.

True end-to-end encryption is possible but would substantially complicate:

- Multi-device synchronization
- Device replacement
- Key recovery
- Message history
- Bots reading questions
- Server-side search
- Adding new devices

End-to-end encryption should be deferred unless protecting messages from the owner’s own server becomes an actual requirement.

## Reliability Requirements

Even at tiny scale, the protocol should handle:

- Temporary network failure
- Reconnecting devices
- Duplicate sends
- Duplicate acknowledgments
- Messages arriving after delay
- Clients missing many messages
- Server restarts
- Removed devices
- Application upgrades

Useful mechanisms include:

- Stable message IDs
- Client-generated idempotency keys
- Monotonically increasing mailbox sequences
- Synchronization cursors
- Acknowledgments
- Transactional persistence
- Protocol versioning
- Explicit error messages

The system does not need distributed consensus if one central server and database remain authoritative.

## Presence and Real-Time State

Presence should be treated as optional and ephemeral.

The server may report that a device or assistant is currently connected, but messages must not depend on presence. Offline recipients receive stored messages when they reconnect.

Typing indicators, live status, and read receipts can be added later if they prove useful.

## Minimal Server Data Model

### `principals`

- `id`
- `stable_name`
- `display_name`
- `type`
- `icon`
- `created_at`

### `devices`

- `id`
- `principal_id`
- `credential_digest`
- `push_subscription`
- `last_seen_at`
- `revoked_at`

### `conversations`

- `id`
- `created_at`

### `conversation_members`

- `conversation_id`
- `principal_id`

### `messages`

- `id`
- `conversation_id`
- `sender_id`
- `sequence`
- `type`
- `payload`
- `client_message_id`
- `created_at`

### `device_cursors`

- `device_id`
- `conversation_id` or `mailbox_id`
- `last_sequence`

### `delivery_receipts`

- `device_id`
- `message_id`
- `delivered_at`
- `read_at`

The first implementation can be simpler if all messages are direct and there is only one primary human.

## Minimal Client

The first client needs only:

- Sign-in or device enrollment
- Inbox
- Conversation view
- Question display
- Answer input
- Synchronization
- Reconnect handling
- Notification permission
- Push-subscription registration
- Basic device management

It does not initially need:

- Public profiles
- Contact discovery
- Group administration
- Media calls
- Public channels
- Reactions
- Stickers
- Social features

## Minimal Implementation Plan

1. Define principals, devices, messages, and mailbox sequences.
2. Define a small versioned JSON message protocol.
3. Implement authentication and device enrollment.
4. Implement WebSocket connection and reconnection.
5. Implement mailbox synchronization.
6. Implement message and answer persistence.
7. Connect assistant questions to protocol messages.
8. Build a small browser/PWA client.
9. Add Web Push as a background wake-up mechanism.
10. Add device revocation and operational status.
11. Retain the current notification mechanism temporarily as a fallback.
12. Consider native clients only after the protocol and PWA are proven.

## Explicit Non-Goals for the Initial Version

Do not initially implement:

- Public registration
- Millions of users
- Federation
- Peer-to-peer networking
- Arbitrary server discovery
- Public contact discovery
- End-to-end encryption
- Group chat
- Audio or video calls
- File sharing
- Social features
- Multiple central servers
- Distributed consensus
- Compatibility with unrelated messaging systems
- A general-purpose public messaging platform

## Feasibility

This is neither impossible nor inherently excessive.

The small user count removes the hardest large-platform problems:

- Horizontal scaling
- Global routing
- Federation
- Spam prevention
- Public abuse moderation
- Complex account recovery
- Broad compatibility

The genuinely difficult pieces are bounded:

- Dependable synchronization
- Authentication and enrollment
- iPhone background delivery
- Client installation and updates
- Protocol evolution
- Device revocation

The server itself can remain small and conventional.

The project is best understood as a private, store-and-forward message bus with human-friendly clients and first-class identities—not as a replacement for TCP/IP, the internet, or Apple’s background-delivery infrastructure.
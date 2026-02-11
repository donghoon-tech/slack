# 📄 Topic 10: Multi-Device Synchronization (Session Management)

> **Prerequisites**: This document addresses the challenge of keeping multiple devices (phone, tablet, desktop) in sync when a user is logged in simultaneously.

## 1. Problem Statement

### 1.1 The Multi-Session Reality
Modern users expect seamless experiences across devices:
*   **Alice** is logged in on Desktop (work), Phone (commute), and Tablet (home).
*   She reads a message on her Phone → Desktop badge must clear **instantly**.
*   She starts typing on Desktop → Phone should show "Draft" (optional).
*   She receives a notification → Only the **inactive** devices should buzz.

### 1.2 The Architectural Challenge
**Question**: When we say "User A is online," do we mean:
*   **Option 1**: User A has **at least one** active connection (any device)?
*   **Option 2**: User A has **N** active connections (track each device separately)?

**Impact**:
*   **Read State Sync**: How do we broadcast "Alice read message X" to all her devices?
*   **Typing Indicators**: Should we show "Alice is typing" if she's typing on Phone but viewing on Desktop?
*   **Push Notifications**: Should we send push to Phone if Desktop is active?

**Goal**: Design a session management system that tracks individual device connections while efficiently synchronizing state across all of a user's devices.

## 2. Solution Strategy Exploration

We analyze three patterns for session management.

### Pattern A: Single Session Per User (Naive)
Treat each user as having exactly one "session," regardless of device count.

*   **Mechanism**: 
    ```
    WebSocket Topic: topic/user/{userId}
    All devices subscribe to the same topic
    ```
*   **Pros**: 
    *   Simple implementation.
    *   Automatic broadcast to all devices.
*   **Cons**: 
    *   **No Device-Level Control**: Can't distinguish which device is active.
    *   **Push Notification Problem**: Can't suppress push for "active" device only.
    *   **Typing Indicator Problem**: User sees "You are typing" on their own other devices.

### Pattern B: Session-Per-Device (Full Tracking)
Track each device connection as a separate session with unique session ID.

*   **Mechanism**: 
    ```
    Session Registry (Redis):
    user:123 → [
      {sessionId: "abc", deviceType: "desktop", gatewayId: "gw-1"},
      {sessionId: "def", deviceType: "mobile", gatewayId: "gw-2"},
      {sessionId: "ghi", deviceType: "tablet", gatewayId: "gw-3"}
    ]
    ```
*   **Pros**: 
    *   **Fine-Grained Control**: Know exactly which devices are active.
    *   **Smart Routing**: Can send events to specific devices (e.g., "typing" to all except sender).
    *   **Push Optimization**: Suppress push for active devices only.
*   **Cons**: 
    *   **Complexity**: Must maintain session registry.
    *   **Lookup Overhead**: Every broadcast requires session lookup.

### Pattern C: Hybrid (User-Topic + Session Metadata)
Use user-level topics for broadcast, but track session metadata for smart decisions.

*   **Mechanism**: 
    ```
    WebSocket Topic: topic/user/{userId} (all devices subscribe)
    Session Metadata (Redis): user:123 → {desktop: active, mobile: active, tablet: inactive}
    
    Decision Logic:
    - Broadcast: Use user topic (simple)
    - Push Notification: Check session metadata (smart)
    - Typing Indicator: Filter on client side
    ```
*   **Pros**: 
    *   **Best of Both Worlds**: Simple broadcast + smart decisions.
    *   **Reduced Lookup**: Only check metadata when needed (e.g., push notifications).
*   **Cons**: 
    *   **Client-Side Filtering**: Some logic pushed to client (e.g., "don't show my own typing").

## 3. Comparative Analysis

| Feature | Pattern A (Single Session) | Pattern B (Session-Per-Device) | Pattern C (Hybrid) |
| :--- | :--- | :--- | :--- |
| **Broadcast Complexity** | 🟢 Very Low | 🔴 High (N lookups) | 🟢 Low |
| **Device-Level Control** | 🔴 None | 🟢 **Full** | 🟡 Partial |
| **Push Optimization** | 🔴 Impossible | 🟢 **Perfect** | 🟢 **Good** |
| **Session Lookup Cost** | 🟢 Zero | 🔴 Every Event | 🟡 Selective |
| **Implementation Complexity** | 🟢 Low | 🔴 High | 🟡 Medium |
| **Slack's Choice** | ❌ (Too naive) | ⚠️ (Partial) | ✅ **(Current)** |

## 4. Proposed Architecture: Hybrid Session Management

We adopt **Pattern C (Hybrid)** for its balance of simplicity and control.

### 4.1 Session Registry Design

**Data Structure** (Redis Hash):
```redis
# Key: session:{userId}
# Value: Hash of {sessionId → metadata}

HSET session:123 "session-abc" '{"deviceType":"desktop","gatewayId":"gw-1","channelId":"456","lastSeen":1707491234567}'
HSET session:123 "session-def" '{"deviceType":"mobile","gatewayId":"gw-2","channelId":"456","lastSeen":1707491234890}'
```

**Why Hash?**
- O(1) lookup for specific session
- O(1) get all sessions for a user (`HGETALL`)
- Automatic cleanup with TTL on parent key

### 4.2 Broadcast Strategy

**For Most Events** (Messages, Read Receipts, Reactions):
```
Use User-Level Topic: topic/user/{userId}
→ All devices receive automatically
→ No session lookup required
```

**For Smart Decisions** (Push Notifications, Typing Indicators):
```
1. Check Session Registry: Is user online in this channel?
2. If yes, which devices are active?
3. Make smart decision (suppress push, filter typing)
```

### 4.3 Implementation Examples

#### Session Registration (On WebSocket Connect)
```java
public void onConnect(WebSocketSession wsSession, Long userId, String deviceType) {
    String sessionId = UUID.randomUUID().toString();
    
    SessionMetadata metadata = SessionMetadata.builder()
        .deviceType(deviceType)
        .gatewayId(getGatewayId())
        .channelId(null) // Set when user enters channel
        .lastSeen(System.currentTimeMillis())
        .build();
    
    // Store in Redis
    redisTemplate.opsForHash().put(
        "session:" + userId,
        sessionId,
        objectMapper.writeValueAsString(metadata)
    );
    
    // Set TTL on parent key (auto-cleanup)
    redisTemplate.expire("session:" + userId, 1, TimeUnit.HOURS);
    
    // Store locally for quick access
    localSessionMap.put(wsSession.getId(), sessionId);
}
```

#### Smart Push Notification (Leveraging Session Data)
```java
public void sendSmartNotification(Message message) {
    for (Long userId : channelMembers) {
        // Step 1: Get all sessions for this user
        Map<Object, Object> sessions = redisTemplate.opsForHash()
            .entries("session:" + userId);
        
        if (sessions.isEmpty()) {
            // User completely offline → Send push
            pushNotificationService.send(userId, message);
            continue;
        }
        
        // Step 2: Check if any session is viewing this channel
        boolean activeInChannel = sessions.values().stream()
            .map(json -> parseMetadata(json))
            .anyMatch(meta -> 
                meta.getChannelId().equals(message.getChannelId()) &&
                (System.currentTimeMillis() - meta.getLastSeen()) < 60_000
            );
        
        if (!activeInChannel) {
            // User online but not viewing this channel → Send push
            pushNotificationService.send(userId, message);
        }
        // else: User is viewing channel → Suppress push
    }
}
```

#### Typing Indicator (Broadcast to Others)
```java
public void handleTypingEvent(Long userId, Long channelId, String sessionId) {
    // Get all sessions for this user
    Map<Object, Object> sessions = redisTemplate.opsForHash()
        .entries("session:" + userId);
    
    // Broadcast to channel members
    TypingEvent event = new TypingEvent(userId, channelId);
    
    // Option 1: Server-side filtering (send to all except sender's sessions)
    for (Long memberId : channelMembers) {
        if (memberId.equals(userId)) {
            // Send only to OTHER sessions of the same user
            Map<Object, Object> userSessions = redisTemplate.opsForHash()
                .entries("session:" + userId);
            
            for (String otherSessionId : userSessions.keySet()) {
                if (!otherSessionId.equals(sessionId)) {
                    sendToSession(otherSessionId, event);
                }
            }
        } else {
            // Send to all sessions of other users
            sendToUser(memberId, event);
        }
    }
    
    // Option 2: Client-side filtering (simpler, send to all)
    // Client checks: if (event.userId === myUserId && event.sessionId === mySessionId) return;
}
```

### 4.4 Synchronization Scenarios

#### Scenario 1: Read State Sync
**Flow**:
```
1. User reads message on Phone
2. Phone → API: markAsRead(userId, channelId, messageId)
3. API → Redis: Update read receipt
4. API → WebSocket: Broadcast to topic/user/{userId}
5. All devices (Desktop, Tablet) receive event → Clear badge
```

**Leverages**: Deep Dive 06 (Read Status Updates)

#### Scenario 2: Typing Indicator
**Flow**:
```
1. User types on Desktop
2. Desktop → API: typing(userId, channelId, sessionId)
3. API → WebSocket: Broadcast to channel members
4. Other users see "Alice is typing"
5. Alice's Phone/Tablet also receive → Client filters out (same userId)
```

**Optimization**: Throttle typing events (max 1 per 3 seconds)

#### Scenario 3: Draft Sync (Optional)
**Flow**:
```
1. User types draft on Desktop
2. Desktop → API: saveDraft(userId, channelId, content) [debounced]
3. API → Redis: Store draft
4. API → WebSocket: Broadcast to topic/user/{userId}
5. Phone/Tablet receive → Show draft indicator
```

**Trade-off**: Network overhead vs UX convenience

## 5. How This System Dies

### Failure Mode 1: Session Registry Stale Data
**Scenario**: User closes laptop lid. WebSocket disconnects, but session entry in Redis hasn't been cleaned up yet.

**Impact**: System thinks user is still "online" on Desktop → Suppresses push notification.

**Mitigation**:
*   **Heartbeat-Based Cleanup**: Update `lastSeen` timestamp on every heartbeat (30s). Consider sessions with `lastSeen > 60s` as stale.
*   **Explicit Disconnect**: WebSocket `onClose` handler removes session from registry immediately.
*   **TTL Fallback**: Redis key expires after 1 hour as safety net.

---

### Failure Mode 2: Session Lookup Latency
**Scenario**: 1000-member channel. Every message triggers 1000 session lookups to decide push notifications.

**Impact**: Redis becomes bottleneck. Latency spikes.

**Why It's Manageable**:
*   Session lookups only needed for **push decisions**, not message broadcast.
*   Can batch lookups using Redis Pipeline.
*   Can cache "user online status" locally for 5-10 seconds.

---

### Failure Mode 3: Redis Unavailability
**Scenario**: Redis cluster goes down. Can't access session registry.

**Impact**: Can't determine which devices are active.

**Decision Tree**:
```
Redis Down → Can't check sessions
  ├─ For Broadcast: Use fallback (send to all known gateways) ✅
  └─ For Push: Fail-open (send push to all devices) ✅
```

**Rationale**: Better to over-notify than miss notifications.

---

### Failure Mode 4: Typing Indicator Storm
**Scenario**: User types rapidly. Client sends 50 typing events in 10 seconds.

**Impact**: Network spam, Redis load.

**Mitigation**:
*   **Client-Side Throttle**: Max 1 typing event per 3 seconds.
*   **Server-Side Dedup**: Ignore typing events if last event was <3s ago.

## 6. Experiment: Session Lookup Performance

### 6.1 Hypothesis
Redis Hash operations (`HGETALL`) can retrieve all sessions for a user in <5ms, even with 10 devices.

### 6.2 Experiment Setup
**Scenario**: Simulate 1000 users, each with 1-5 active sessions.

**Operations**:
1. `HSET` to register sessions (write)
2. `HGETALL` to retrieve all sessions for a user (read)
3. `HDEL` to remove session on disconnect

**Metrics**:
- Latency P95 for `HGETALL`
- Throughput (ops/sec)

### 6.3 Implementation
```
experiments/session-lookup-bench/
├── register-sessions.js    # Populate Redis with sessions
├── lookup-benchmark.js     # Measure HGETALL performance
└── cleanup-test.js         # Test TTL and HDEL
```

**Expected Result**: P95 <5ms for session lookup, validating hybrid approach.

## 7. Verdict & Roadmap

*   **Decision**: Use **Hybrid Session Management (Pattern C)**.
*   **Session Storage**: Redis Hash with TTL-based cleanup.
*   **Broadcast**: User-level topics for most events.
*   **Smart Decisions**: Session lookup for push notifications and typing indicators.
*   **Next Steps**:
    1.  Implement session registry with heartbeat-based cleanup.
    2.  Integrate with push notification system (Deep Dive 09).
    3.  Add typing indicator with throttling.
    4.  Run session lookup benchmark.

## 8. Related Topics

*   **Massive Fan-out**: Session registry concept introduced.
    *   **→ See Deep Dive 03**
*   **Gateway Separation**: Session lookup for routing.
    *   **→ See Deep Dive 04**
*   **Read Status Updates**: Foundation for read state sync.
    *   **→ See Deep Dive 06**
*   **Smart Notifications**: Presence-based push suppression.
    *   **→ See Deep Dive 09**

## 9. Architectural Decision Records

*   **ADR-11**: Multi-Device Session Management (To be created)

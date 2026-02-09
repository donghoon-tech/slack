# 📄 Topic 09: Smart Notification Routing (Push Notifications)

> **Prerequisites**: This document addresses the challenge of minimizing notification fatigue and infrastructure costs while maintaining timely delivery of critical alerts.

## 1. Problem Statement

### 1.1 The Notification Fatigue Problem
Sending a push notification (FCM/APNS) for *every* message creates a terrible user experience:
*   **Spam**: User A is actively chatting in a channel. They receive 50 push notifications for messages they're already reading in real-time.
*   **Battery Drain**: Constant push notifications wake up mobile devices, draining battery and consuming data.
*   **User Annoyance**: Users disable notifications entirely, defeating the purpose of the feature.

### 1.2 The Cost Problem
Push notifications are not free:
*   **FCM/APNS Costs**: While FCM is free for basic usage, APNS and high-volume FCM usage incur costs. More importantly, unnecessary pushes waste server resources (CPU, network).
*   **Scale**: 1,000,000 users × 100 messages/day = **100M potential push notifications/day**. If 80% are unnecessary (user is online), we're wasting **80M pushes/day**.

### 1.3 The Distributed Challenge
In a multi-server environment, how do we answer these questions with low latency?
*   **Presence Check**: "Is User A currently online in Channel X?" (Need answer in <10ms to avoid delaying message delivery)
*   **Throttling**: "Did User B already receive a push for this channel in the last 10 seconds?" (Need distributed lock across multiple servers)

**Goal**: Design a notification system that **suppresses** pushes for online users, **throttles** rapid-fire notifications, and **guarantees delivery** for offline users—all while maintaining sub-10ms decision latency.

## 2. Solution Strategy Exploration

We analyze four patterns for smart notification routing.

### Pattern A: Always Send (Naive)
Send a push notification for every message to every channel member.
*   **Pros**: Simple, guaranteed delivery.
*   **Cons**: **Notification Fatigue**. Users will disable notifications. **Cost Waste**. 80% of pushes are unnecessary.

### Pattern B: Client-Side Suppression
Mobile app checks if it's in the foreground and discards the notification.
*   **Pros**: Simple server logic.
*   **Cons**: **Wasted Resources**. Push is still sent over the network, consuming server bandwidth and FCM/APNS quota. **Battery Drain**. Device still wakes up to process the notification.

### Pattern C: Server-Side Presence Check (Redis ZSET)
Before sending a push, query the presence system to check if the user is online.
*   **Mechanism**: 
    ```java
    if (!presenceService.isUserOnlineInChannel(userId, channelId)) {
        pushNotificationService.send(userId, message);
    }
    ```
*   **Pros**: **Effective Suppression**. Eliminates 70-80% of unnecessary pushes. **Cost Savings**. Reduces FCM/APNS usage and server load.
*   **Cons**: **Latency Risk**. If presence check is slow (>50ms), it delays message delivery. **False Negatives**. If presence data is stale (user just went offline), they might miss a notification.

### Pattern D: Server-Side Presence Check + Throttling (Redis Lock)
Combine presence check with a distributed throttling mechanism to prevent notification spam.
*   **Mechanism**: 
    ```java
    // Step 1: Check if user is online
    if (presenceService.isUserOnlineInChannel(userId, channelId)) {
        return; // Suppress notification
    }
    
    // Step 2: Check if we already sent a notification recently
    String lockKey = "push_throttle:" + userId + ":" + channelId;
    if (redisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS)) {
        // Lock acquired, send notification
        pushNotificationService.send(userId, createSummary(channelId));
    }
    // Lock not acquired = notification already sent in last 10s, skip
    ```
*   **Pros**: 
    *   **Maximum Suppression**: Eliminates 70-80% via presence check + additional 10-15% via throttling.
    *   **Better UX**: User receives at most 1 push per channel per 10 seconds, even if 50 messages arrive.
    *   **Distributed Safety**: Redis lock works across multiple servers.
*   **Cons**: **Complexity**. Requires Redis integration and careful lock management.

## 3. Comparative Analysis

| Feature | Pattern A (Always Send) | Pattern B (Client-Side) | Pattern C (Presence Check) | Pattern D (Presence + Throttle) |
| :--- | :--- | :--- | :--- | :--- |
| **User Experience** | 🔴 Terrible (Spam) | 🔴 Bad (Battery Drain) | 🟢 Good | 🟢 **Excellent** |
| **Cost Efficiency** | 🔴 Wasteful | 🔴 Wasteful | 🟢 Good (70-80% savings) | 🟢 **Best (85-90% savings)** |
| **Latency** | 🟢 Low | 🟢 Low | 🟡 Medium (Presence Query) | 🟡 Medium (Presence + Lock) |
| **Complexity** | 🟢 Very Low | 🟢 Low | 🟡 Medium | 🔴 High |
| **Distributed Safety** | 🟢 N/A | 🟢 N/A | 🟢 Stateless | 🟢 **Redis Lock** |
| **Slack's Choice** | ❌ (Never) | ❌ (Early Days) | ⚠️ (Partial) | ✅ **(Current)** |

## 4. Proposed Architecture: Presence Check + Throttling

We adopt **Pattern D (Presence Check + Throttling)** for maximum efficiency and UX quality.

### 4.1 Decision Flow
```
1. Message arrives at server
2. Server publishes to WebSocket (real-time delivery)
3. For each channel member:
   a. Check Presence: Is user online in this channel?
      - YES → Skip push notification
      - NO → Continue to step b
   b. Check Throttle Lock: Did we send a push in last 10s?
      - LOCK EXISTS → Skip (already notified)
      - LOCK ACQUIRED → Send push notification
4. Push notification sent with summary: "5 new messages in #general"
```

### 4.2 Implementation Details

#### Presence Check (Leveraging Deep Dive 07)
```java
public boolean isUserOnlineInChannel(Long userId, Long channelId) {
    String presenceKey = "presence:" + channelId;
    Double score = redisTemplate.opsForZSet().score(presenceKey, userId.toString());
    
    if (score == null) return false;
    
    long lastSeen = score.longValue();
    long now = System.currentTimeMillis();
    return (now - lastSeen) < 60_000; // Online if heartbeat within 60s
}
```

#### Throttle Lock (Redis SETNX)
```java
public boolean acquirePushLock(Long userId, Long channelId, int ttlSeconds) {
    String lockKey = "push_throttle:" + userId + ":" + channelId;
    Boolean acquired = redisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", ttlSeconds, TimeUnit.SECONDS);
    return Boolean.TRUE.equals(acquired);
}
```

#### Notification Service Integration
```java
@Async
public void sendSmartNotification(Message message) {
    List<Long> channelMembers = channelService.getMembers(message.getChannelId());
    
    for (Long userId : channelMembers) {
        // Skip sender
        if (userId.equals(message.getSenderId())) continue;
        
        // Step 1: Presence suppression
        if (isUserOnlineInChannel(userId, message.getChannelId())) {
            metrics.increment("push.suppressed.online");
            continue;
        }
        
        // Step 2: Throttle check
        if (!acquirePushLock(userId, message.getChannelId(), 10)) {
            metrics.increment("push.suppressed.throttle");
            continue;
        }
        
        // Step 3: Send notification
        String summary = buildSummary(userId, message.getChannelId());
        pushNotificationService.send(userId, summary);
        metrics.increment("push.sent");
    }
}
```

### 4.3 Notification Summary Strategy
Instead of sending the full message content, send a summary:
*   **Single Message**: "Alice: Hey, are you free?" (Show message preview)
*   **Multiple Messages (Throttled)**: "5 new messages in #general" (Show count)
*   **High-Priority**: "@mention" or "Direct Message" → Always send immediately (bypass throttle)

## 5. How This System Dies

### Failure Mode 1: Stale Presence Data (The "Ghost Online" Problem)
**Scenario**: User closes laptop lid. WebSocket disconnects, but presence system hasn't detected it yet (within 60s heartbeat window).

**Impact**: User misses critical notifications for up to 60 seconds.

**Why It Happens**:
*   Heartbeat interval (30-60s) creates a detection lag.
*   Network partitions can delay disconnect detection.
*   Server crashes prevent explicit disconnect events.

**Mitigation**:
*   **Fail-Safe for Critical Messages**: @mentions and DMs bypass presence check entirely.
*   **Explicit Disconnect**: Mobile apps send disconnect event when backgrounded.
*   **Shorter Heartbeat**: Reduce to 30s (trade-off: 2x Redis write load).

---

### Failure Mode 2: Redis Unavailability (The "Blind Decision" Problem)
**Scenario**: Redis cluster goes down. Presence checks fail.

**Impact**: System can't determine who's online.

**Decision Tree**:
```
Redis Down → Can't check presence
  ├─ Fail-Closed: Don't send any pushes → Users miss notifications ❌
  └─ Fail-Open: Send all pushes → Notification spam ✅ (Lesser evil)
```

**Why Fail-Open Wins**: Missing a notification is worse than receiving an extra one.

---

### Failure Mode 3: Lock Contention (The "Thundering Herd" Problem)
**Scenario**: 1000 messages/sec in a single channel. Every message tries to acquire throttle lock for the same user.

**Impact**: Redis CPU spikes. Lock acquisition latency increases.

**Why It's Actually Fine**:
*   Locks are **per-user-per-channel**, not global.
*   1000 messages → 1000 different users → naturally distributed.
*   Even if contention occurs, SETNX is O(1) and fast (<3ms).

---

### Failure Mode 4: The "Just Went Offline" Race Condition
**Scenario**:
```
T0: User closes app
T1: Message arrives, presence check → "Online" (stale data)
T2: Push suppressed
T3: Presence expires
Result: User never gets notified
```

**Probability**: Low (requires message to arrive in the 1-2s window between disconnect and presence expiration).

**Mitigation**: Acceptable trade-off. Alternative (always send push) causes 80% spam.

## 6. Experiment: Suppression Effectiveness (Notification Lab)

### 6.1 Hypothesis
Presence-based suppression + throttling can reduce push notifications by **80-90%** without missing critical alerts.

### 6.2 Experiment Setup
**Scenario**: Simulate 100 users in a channel receiving 100 messages over 10 seconds.

**User Distribution**:
*   **50 Active** (online, viewing channel) → Should receive 0 pushes
*   **30 Idle** (online, not viewing) → Should receive pushes
*   **20 Offline** → Should receive pushes

**Message Pattern**:
*   Normal: 1 message/sec
*   Burst: 10 messages in 1 second (test throttling)

### 6.3 What We Measure
1. **Suppression Rate**: % of pushes avoided (target: 80%+)
2. **False Negatives**: Offline users who didn't get notified (target: 0%)
3. **Throttle Effectiveness**: Burst → 1 push per user (not 10)

### 6.4 Implementation
```
experiments/notification-lab/
├── simulate-presence.js      # Mock Redis presence data
├── simulate-messages.js      # Generate message events
├── smart-notification.js     # Pattern D implementation
└── measure-suppression.js    # Count sent vs suppressed
```

**Expected Result**: ~80% suppression (50 active users × 100 messages = 5000 suppressed out of 10000 total).

## 7. Risks & Mitigations

### Risk 1: Stale Presence Data
*   **Impact**: User goes offline, but presence system hasn't detected it yet (within 60s window). They miss a notification.
*   **Mitigation**: 
    *   Reduce heartbeat interval to 30s (trade-off: more Redis writes).
    *   Implement "Critical Message" flag (e.g., @mentions) that bypasses presence check.
    *   Mobile apps send explicit "Disconnect" event when going to background.

### Risk 2: Redis Failure
*   **Impact**: Presence checks fail, system can't determine online status.
*   **Mitigation**: 
    *   **Fail-Open**: If Redis is unavailable, assume user is offline and send push (better to over-notify than miss).
    *   Redis replication (master-slave) for high availability.

### Risk 3: Lock Contention
*   **Impact**: High message volume in a single channel causes lock contention.
*   **Mitigation**: 
    *   Locks are per-user-per-channel, so contention is naturally distributed.
    *   Use short TTL (10s) to minimize lock duration.

### Risk 4: Notification Delay
*   **Impact**: Throttling causes a 10-second delay for the second notification.
*   **Mitigation**: 
    *   This is intentional (UX improvement). Users prefer 1 summary notification over 10 individual ones.
    *   For critical messages (@mentions, DMs), bypass throttle.

## 8. Verdict & Roadmap

*   **Decision**: Use **Presence Check + Throttling (Pattern D)** for production notifications.
*   **Experiment**: Build `notification-bench` to validate suppression rates and latency.
*   **Next Steps**:
    1.  Implement `SmartNotificationService` with presence integration.
    2.  Run benchmark to measure suppression effectiveness.
    3.  Integrate with FCM/APNS for real push delivery.
    4.  Monitor metrics: `push.sent`, `push.suppressed.online`, `push.suppressed.throttle`.

## 9. Related Topics

*   **Distributed Presence System**: Foundation for online/offline detection.
    *   **→ See Deep Dive 07**
*   **Redis Data Structures**: ZSET for presence, SETNX for locks.
    *   **→ See ADR-07**
*   **Event-Driven Architecture**: Async notification processing.
    *   **→ See ADR-04**

## 10. Architectural Decision Records

*   **ADR-10**: Smart Notification Routing with Presence-Based Suppression (To be created)

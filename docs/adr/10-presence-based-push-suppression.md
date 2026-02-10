# ADR-0010: Presence-Based Push Notification Suppression

## Metadata

- **Status**: Proposed 📝
- **Date**: 2026-02-09
- **Context**: Future - Mobile Push Notification Optimization
- **Related Deep Dive**: [Deep Dive 09: Smart Notification Routing](../deepdives/09-smart-notifications.md)
- **Related ADR**: [ADR-0007: Redis ZSET for Presence](./07-redis-zset-presence.md)

---

## TL;DR (Executive Summary)

**Decision**: Use **Presence-Based Suppression + Throttling** to reduce push notifications by 85-90%.

**Key Trade-off**: Accept potential notification miss during presence detection lag (1-60s) in exchange for **massive cost savings** and **better user experience**.

**Rationale**: Sending push notifications for every message creates notification fatigue and wastes resources. By checking if a user is online (via presence system) before sending a push, we eliminate 70-80% of unnecessary notifications. Adding throttling (max 1 push per channel per 10s) eliminates another 10-15%, while preventing notification spam during message bursts.

---

## Context

### The Problem

1. **Notification Fatigue**: Users actively chatting receive push notifications for messages they're already reading in real-time.
2. **Cost Waste**: 1M users × 100 messages/day = 100M potential pushes. If 80% are unnecessary (user is online), we waste 80M pushes/day.
3. **Battery Drain**: Constant push notifications wake mobile devices, draining battery.

### Constraints

- **Decision Latency**: Must determine "send push or not" in <10ms to avoid delaying message delivery.
- **Zero False Negatives**: Offline users must ALWAYS receive notifications (fail-open).
- **Distributed Safety**: Throttling must work across multiple servers.

---

## Decision

### What We Chose

**Two-Stage Suppression**:

1. **Presence Check**: Query Redis ZSET to check if user is online in the channel.
2. **Throttle Lock**: Use Redis SETNX to ensure max 1 push per user per channel per 10 seconds.

### Architecture Flow

```
Message Arrives
  ↓
WebSocket Broadcast (Real-time)
  ↓
For each channel member:
  ├─ Step 1: Presence Check (Redis ZSCORE)
  │   ├─ Online? → Skip push ✅
  │   └─ Offline? → Continue to Step 2
  ├─ Step 2: Throttle Lock (Redis SETNX)
  │   ├─ Lock exists? → Skip (already notified) ✅
  │   └─ Lock acquired? → Send push 📱
```

### Why This Choice

| Strategy | Suppression Rate | Latency | Complexity | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Always Send** | 0% | Low | Very Low | ❌ Rejected (Spam) |
| **Client-Side** | ~30% | Low | Low | ❌ Rejected (Wastes network/battery) |
| **Presence Only** | 70-80% | <5ms | Medium | ⚠️ Partial |
| **Presence + Throttle** | **85-90%** | **<10ms** | High | ✅ **Selected** |

**Key Insight**: Client-side suppression still sends the push over the network and wakes the device. Server-side suppression prevents the push from being sent at all.

---

## Consequences

### Positive Impacts

✅ **Cost Savings**: 85-90% reduction in FCM/APNS API calls and network bandwidth.

✅ **Better UX**: Users receive at most 1 push per channel per 10s, even during message bursts.

✅ **Battery Efficiency**: Devices aren't woken unnecessarily.

✅ **Distributed Safety**: Redis lock works across multiple application servers.

### Negative Impacts & Mitigations

❌ **Risk: Stale Presence Data**
- **Impact**: User goes offline, but presence hasn't expired yet (within 60s window). Push is suppressed, user misses notification.
- **Probability**: Low (requires message to arrive in 1-2s window after disconnect).
- **Mitigation**:
  - **Critical Messages Bypass**: @mentions and DMs skip presence check entirely.
  - **Explicit Disconnect**: Mobile apps send disconnect event when backgrounded.
  - **Shorter Heartbeat**: Reduce to 30s (trade-off: 2x Redis write load).

❌ **Risk: Redis Unavailability**
- **Impact**: Can't check presence or acquire locks.
- **Mitigation**:
  - **Fail-Open**: If Redis is down, assume user is offline and send all pushes.
  - **Rationale**: Missing a notification is worse than receiving an extra one.

❌ **Risk: Notification Delay**
- **Impact**: Throttling delays the 2nd+ notification by up to 10 seconds.
- **Mitigation**:
  - **Intentional UX**: Users prefer 1 summary ("5 new messages") over 10 individual pushes.
  - **Critical Messages Bypass**: @mentions and DMs skip throttle.

### Implementation Details

**Presence Check** (Leverages ADR-07):
```java
public boolean isUserOnlineInChannel(Long userId, Long channelId) {
    String presenceKey = "presence:channel:" + channelId;
    Double score = redisTemplate.opsForZSet().score(presenceKey, userId.toString());
    
    if (score == null) return false;
    
    long lastSeen = score.longValue();
    long now = System.currentTimeMillis();
    return (now - lastSeen) < 60_000; // 60s threshold
}
```

**Throttle Lock**:
```java
public boolean acquirePushLock(Long userId, Long channelId, int ttlSeconds) {
    String lockKey = "push_throttle:" + userId + ":" + channelId;
    return Boolean.TRUE.equals(
        redisTemplate.opsForValue().setIfAbsent(lockKey, "1", ttlSeconds, TimeUnit.SECONDS)
    );
}
```

**Integrated Service**:
```java
@Async
public void sendSmartNotification(Message message) {
    for (Long userId : channelMembers) {
        // Step 1: Presence suppression
        if (isUserOnlineInChannel(userId, channelId)) {
            metrics.increment("push.suppressed.online");
            continue;
        }
        
        // Step 2: Throttle check
        if (!acquirePushLock(userId, channelId, 10)) {
            metrics.increment("push.suppressed.throttle");
            continue;
        }
        
        // Step 3: Send push
        pushNotificationService.send(userId, buildSummary(message));
        metrics.increment("push.sent");
    }
}
```

---

## Alternatives Considered

### Alternative 1: Always Send (Pattern A)
**Approach**: Send push for every message.

**Why Rejected**: Creates notification fatigue. Users disable notifications entirely.

### Alternative 2: Client-Side Suppression (Pattern B)
**Approach**: Send push to device, let app decide whether to show it.

**Why Rejected**: 
- Push is still sent over network (wastes bandwidth).
- Device still wakes up (wastes battery).
- FCM/APNS API still called (costs money).

### Alternative 3: Presence Check Only (Pattern C)
**Approach**: Check presence, but no throttling.

**Why Not Primary Choice**: 
- Doesn't handle message bursts well.
- User receives 10 pushes if 10 messages arrive while offline.
- Pattern D (Presence + Throttle) provides better UX with minimal additional complexity.

---

## Success Metrics

**Before (No Suppression)**:
- Push notifications sent: 100M/day
- User complaints: High (notification spam)
- Battery impact: High

**After (Presence + Throttle)**:
- Push notifications sent: 10-15M/day (85-90% reduction)
- Suppression breakdown:
  - Presence: 70-80%
  - Throttle: 10-15%
- False negative rate: 0% (all offline users notified)
- Decision latency: P95 <10ms

**Measure**:
- `push.sent` (total pushes sent)
- `push.suppressed.online` (presence-based)
- `push.suppressed.throttle` (lock-based)
- `push.latency` (decision time)

---

## Related Decisions

- **[ADR-0007: Redis ZSET for Presence](./07-redis-zset-presence.md)**: Foundation for online/offline detection
- **[ADR-0004: Event-Driven Architecture](./04-event-driven-architecture.md)**: Async notification processing

---

## References

- **[Deep Dive 09: Smart Notification Routing](../deepdives/09-smart-notifications.md)**: Detailed analysis and failure modes
- Slack Engineering: Desktop-first notification strategy
- Discord Engineering: ML-based notification optimization

---

## Notes

**Why Fail-Open on Redis Failure?**

When Redis is unavailable, we have two choices:
1. **Fail-Closed**: Don't send any pushes → Users miss critical notifications ❌
2. **Fail-Open**: Send all pushes → Temporary notification spam ✅

We choose fail-open because missing a notification is worse than receiving an extra one. Users can tolerate temporary spam during an outage, but missing a critical @mention is unacceptable.

**Critical Message Bypass**:
- @mentions: Always send immediately (skip presence + throttle)
- Direct Messages: Always send immediately
- Regular messages: Subject to suppression

This ensures important notifications are never missed while still reducing overall notification volume by 85-90%.

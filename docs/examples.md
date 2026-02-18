# Examples

This page provides complete, real-world examples of using dictrack.

## Example 1: User Activity Tracker

Track user activities and notify when they complete certain milestones.

```python
"""
User Activity Tracker Example

Track user activities and notify when users reach activity milestones.
"""

from dictrack.conditions.keys import KeyValueEQ, KeyValueInList
from dictrack.data_caches.redis import RedisDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.events import EVENT_TRACKER_ALL_COMPLETED, EVENT_TRACKER_STAGE_COMPLETED, TrackerEvent
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker


def setup_event_handlers(manager):
    """Set up event handlers for tracking milestones."""
    
    def on_stage_completed(event):
        print(f"🎯 Milestone reached! User {event.group_id} completed stage: {event.tracker.progress}/{event.tracker.target}")
    
    def on_all_completed(event):
        print(f"🎉 User {event.group_id} completed all milestones! Target: {event.tracker.target}")
    
    manager.add_listener(EVENT_TRACKER_STAGE_COMPLETED, on_stage_completed)
    manager.add_listener(EVENT_TRACKER_ALL_COMPLETED, on_all_completed)


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=RedisDataCache(host="localhost", port=6379, db=0),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="user_tracking"),
    )
    
    # Set up event handlers
    setup_event_handlers(manager)
    
    # Create milestones for users
    milestones = [10, 50, 100, 500, 1000]
    
    for user_id in ["user_001", "user_002", "user_003"]:
        tracker = CountTracker(
            name="activity_milestone",
            target=milestones,
            conditions=[KeyValueInList(key="event_type", value=["login", "purchase", "comment", "share"])],
        )
        manager.add_tracker(group_id=user_id, tracker=tracker)
    
    # Simulate user activities
    activities = [
        {"user_id": "user_001", "event_type": "login", "timestamp": 1699000000},
        {"user_id": "user_001", "event_type": "purchase", "timestamp": 1699000100},
        {"user_id": "user_001", "event_type": "comment", "timestamp": 1699000200},
    ]
    
    for activity in activities:
        # Track only if the user_id matches and event_type is valid
        is_ok, dirtied, completed, limited = manager.track(
            group_id=activity["user_id"],
            data=activity,
        )
        
        print(f"Tracked: {activity['event_type']} for {activity['user_id']}")
        for tracker in dirtied:
            print(f"  Progress: {tracker.progress}/{tracker.target}")


if __name__ == "__main__":
    main()
```

## Example 2: Rate Limiter

Implement rate limiting for API requests.

```python
"""
Rate Limiter Example

Implement rate limiting using CountLimiter.
"""

from dictrack.conditions.keys import KeyValueEQ
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.limiters.count import CountLimiter
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=MemoryDataCache(),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="rate_limiter"),
    )
    
    # Create rate limiter: 100 requests per minute
    rate_limiter = CountLimiter(count=100)
    
    # Create tracker with rate limiter
    tracker = CountTracker(
        name="api_rate_limit",
        target=1000,  # High target, but limiter will stop at 100
        conditions=[KeyValueEQ(key="endpoint", value="/api/v1/data")],
        limiters=[rate_limiter],
    )
    
    manager.add_tracker(group_id="api_client_001", tracker=tracker)
    
    # Simulate API requests
    print("Starting rate limiter test...")
    for i in range(105):
        data = {"endpoint": "/api/v1/data", "request_id": i, "client": "api_client_001"}
        
        is_ok, dirtied, completed, limited = manager.track(
            group_id="api_client_001",
            data=data,
        )
        
        if limited:
            print(f"❌ Request {i} blocked! Rate limit exceeded.")
            print(f"   Limiter remaining: {rate_limiter.remaining}")
            break
        else:
            print(f"✅ Request {i} allowed. Remaining: {rate_limiter.remaining}")
    
    # Reset limiter for next minute
    print("\nResetting rate limiter...")
    manager.reset_tracker(
        group_id="api_client_001",
        name="api_rate_limit",
        reset_policy=ResetPolicy.LIMITER,
    )


if __name__ == "__main__":
    from dictrack.trackers import ResetPolicy
    main()
```

## Example 3: Order Processing System

Track order processing with multiple stages.

```python
"""
Order Processing System Example

Track order processing stages with conditions.
"""

from dictrack.conditions.keys import KeyValueEQ, KeyValueInList
from dictrack.data_caches.redis import RedisDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.events import EVENT_TRACKER_ALL_COMPLETED, EVENT_TRACKER_STAGE_COMPLETED, TrackerEvent
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.accumulation import AccumulationTracker
from dictrack.trackers.numerics.count import CountTracker


def setup_order_tracking(manager):
    """Set up order tracking with event handlers."""
    
    def on_order_completed(event):
        print(f"✅ Order {event.name} fully processed! Total amount: ${event.tracker.progress}")
    
    def on_stage_completed(event):
        print(f"📦 Order {event.name} stage completed: ${event.tracker.progress}/${event.tracker.target}")
    
    manager.add_listener(EVENT_TRACKER_ALL_COMPLETED, on_order_completed)
    manager.add_listener(EVENT_TRACKER_STAGE_COMPLETED, on_stage_completed)


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=RedisDataCache(host="localhost", port=6379, db=1),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="order_system"),
    )
    
    setup_order_tracking(manager)
    
    # Track order count (number of orders)
    order_count_tracker = CountTracker(
        name="order_count",
        target=[10, 50, 100, 500],
        conditions=[KeyValueEQ(key="status", value="completed")],
    )
    
    # Track order amount (total revenue)
    amount_tracker = AccumulationTracker(
        name="order_amount",
        target=[1000, 5000, 10000, 50000],
        conditions=[
            KeyValueEQ(key="status", value="completed"),
            KeyValueInList(key="payment_method", value=["credit_card", "debit_card", "paypal"]),
        ],
    )
    
    # Add trackers
    manager.add_trackers(group_id="orders", trackers=[order_count_tracker, amount_tracker])
    
    # Simulate orders
    orders = [
        {"order_id": "ORD001", "status": "completed", "amount": 150.00, "payment_method": "credit_card"},
        {"order_id": "ORD002", "status": "pending", "amount": 200.00, "payment_method": "paypal"},
        {"order_id": "ORD003", "status": "completed", "amount": 75.50, "payment_method": "debit_card"},
        {"order_id": "ORD004", "status": "completed", "amount": 320.00, "payment_method": "credit_card"},
        {"order_id": "ORD005", "status": "cancelled", "amount": 50.00, "payment_method": "credit_card"},
    ]
    
    for order in orders:
        print(f"\nProcessing order: {order['order_id']}")
        
        # Track order count
        is_ok, dirtied, completed, limited = manager.track(
            group_id="orders",
            data=order,
        )
        
        # Check results
        trackers = manager.get_trackers(group_id="orders")
        for tracker in trackers:
            if tracker.name == "order_count":
                print(f"  Orders: {tracker.progress}/{tracker.target}")
            elif tracker.name == "order_amount":
                print(f"  Revenue: ${tracker.progress}/${tracker.target}")


if __name__ == "__main__":
    main()
```

## Example 4: Campaign Analytics

Track marketing campaign performance.

```python
"""
Campaign Analytics Example

Track campaign clicks, conversions, and revenue.
"""

from dictrack.conditions.keys import KeyValueEQ, KeyValueInList, KeyValueGT
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker
from dictrack.trackers.numerics.accumulation import AccumulationTracker


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=MemoryDataCache(),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="campaign_analytics"),
    )
    
    # Campaign click tracker
    click_tracker = CountTracker(
        name="campaign_clicks",
        target=[1000, 5000, 10000],
        conditions=[KeyValueEQ(key="event", value="click")],
    )
    
    # Campaign conversion tracker (only count clicks with conversion)
    conversion_tracker = CountTracker(
        name="campaign_conversions",
        target=[100, 500, 1000],
        conditions=[
            KeyValueEQ(key="event", value="click"),
            KeyValueEQ(key="converted", value=True),
        ],
    )
    
    # Revenue tracker
    revenue_tracker = AccumulationTracker(
        name="campaign_revenue",
        target=[5000, 25000, 50000],
        conditions=[
            KeyValueEQ(key="event", value="purchase"),
            KeyValueGT(key="amount", value=0),
        ],
    )
    
    # Add all trackers
    manager.add_trackers(
        group_id="campaign_summer_2024",
        trackers=[click_tracker, conversion_tracker, revenue_tracker],
    )
    
    # Simulate campaign events
    events = [
        {"event": "click", "converted": False, "amount": 0},
        {"event": "click", "converted": True, "amount": 0},
        {"event": "click", "converted": False, "amount": 0},
        {"event": "purchase", "converted": True, "amount": 50.00},
        {"event": "click", "converted": False, "amount": 0},
        {"event": "purchase", "converted": True, "amount": 125.00},
        {"event": "click", "converted": True, "amount": 0},
        {"event": "purchase", "converted": True, "amount": 75.00},
    ]
    
    print("Campaign Analytics - Summer 2024")
    print("=" * 50)
    
    for event in events:
        is_ok, dirtied, completed, limited = manager.track(
            group_id="campaign_summer_2024",
            data=event,
        )
        
        print(f"Event: {event['event']}", end="")
        if event.get('converted'):
            print(" (converted)", end="")
        if event.get('amount', 0) > 0:
            print(f" - ${event['amount']:.2f}", end="")
        print()
    
    # Get final stats
    trackers = manager.get_trackers(group_id="campaign_summer_2024")
    
    print("\n" + "=" * 50)
    print("Final Campaign Stats:")
    for tracker in trackers:
        print(f"  {tracker.name}: {tracker.progress}/{tracker.target}")


if __name__ == "__main__":
    main()
```

## Example 5: Session Monitoring

Monitor user sessions with time-based limiting.

```python
"""
Session Monitoring Example

Monitor user sessions with time-based limiting.
"""

import time
from dictrack.conditions.keys import KeyValueEQ
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.limiters.time import TimeLimiter
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker
from dictrack.trackers import ResetPolicy


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=MemoryDataCache(),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="session_monitoring"),
    )
    
    # Create session tracker with time limiter (60 seconds window)
    session_tracker = CountTracker(
        name="session_actions",
        target=100,
        conditions=[KeyValueEQ(key="action", value="api_call")],
        limiters=[TimeLimiter(seconds=60)],
    )
    
    # Add tracker for user
    manager.add_tracker(group_id="user_session_001", tracker=session_tracker)
    
    # Simulate API calls
    print("Simulating API calls for 70 seconds...")
    print("-" * 40)
    
    start_time = time.time()
    call_count = 0
    
    for i in range(70):
        # Check if we're in a new second
        current_time = time.time()
        elapsed = current_time - start_time
        
        # Make API call
        data = {"action": "api_call", "endpoint": "/api/data", "call_id": i}
        
        is_ok, dirtied, completed, limited = manager.track(
            group_id="user_session_001",
            data=data,
        )
        
        if not limited:
            call_count += 1
            print(f"Second {int(elapsed)}: ✅ API call #{call_count} allowed")
        else:
            print(f"Second {int(elapsed)}: ❌ API call blocked - time window exceeded")
            print(f"  Time limiter will reset at: {int(elapsed) + 60} seconds")
        
        # Check if we should reset the limiter (every 60 seconds)
        if elapsed > 0 and int(elapsed) % 60 == 0 and elapsed > 60:
            print(f"\n--- Resetting time limiter at {int(elapsed)} seconds ---")
            manager.reset_tracker(
                group_id="user_session_001",
                name="session_actions",
                reset_policy=ResetPolicy.LIMITER,
            )
        
        time.sleep(1)
    
    print("\n" + "-" * 40)
    print(f"Total API calls made: {call_count}")


if __name__ == "__main__":
    main()
```

## Example 6: Custom Condition

Create a custom condition for specific business logic.

```python
"""
Custom Condition Example

Create a custom condition for business-specific logic.
"""

from dictrack.conditions.base import BaseCondition
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker


class HighValueOrderCondition(BaseCondition):
    """
    Custom condition: Track only high-value orders.
    
    Rules:
    - Order amount >= 100
    - Customer tier in [gold, platinum]
    - Not from excluded regions
    """
    
    def __init__(self, min_amount=100, allowed_tiers=None, excluded_regions=None):
        self._min_amount = min_amount
        self._allowed_tiers = allowed_tiers or ["gold", "platinum"]
        self._excluded_regions = excluded_regions or ["blocked_region"]
    
    def __repr__(self):
        return "<HighValueOrderCondition (min_amount={} tiers={})>".format(
            self._min_amount, self._allowed_tiers
        )
    
    def check(self, data, *args, **kwargs):
        # Check minimum amount
        amount = data.get("amount", 0)
        if amount < self._min_amount:
            return False
        
        # Check customer tier
        tier = data.get("customer_tier", "").lower()
        if tier not in self._allowed_tiers:
            return False
        
        # Check excluded regions
        region = data.get("region", "").lower()
        if region in self._excluded_regions:
            return False
        
        return True


def main():
    # Initialize manager
    manager = TrackingManager(
        data_cache=MemoryDataCache(),
        data_store=MongoDBDataStore(host="localhost", port=27017, database="order_tracking"),
    )
    
    # Create tracker with custom condition
    tracker = CountTracker(
        name="high_value_orders",
        target=[10, 50, 100],
        conditions=[
            HighValueOrderCondition(
                min_amount=100,
                allowed_tiers=["gold", "platinum"],
                excluded_regions=["blocked", "suspended"],
            )
        ],
    )
    
    manager.add_tracker(group_id="orders", tracker=tracker)
    
    # Test orders
    orders = [
        {"order_id": "ORD001", "amount": 50, "customer_tier": "gold", "region": "us"},
        {"order_id": "ORD002", "amount": 150, "customer_tier": "gold", "region": "us"},
        {"order_id": "ORD003", "amount": 200, "customer_tier": "platinum", "region": "eu"},
        {"order_id": "ORD004", "amount": 120, "customer_tier": "silver", "region": "us"},
        {"order_id": "ORD005", "amount": 300, "customer_tier": "gold", "region": "blocked"},
        {"order_id": "ORD006", "amount": 180, "customer_tier": "platinum", "region": "uk"},
    ]
    
    print("Processing Orders:")
    print("-" * 60)
    
    for order in orders:
        is_ok, dirtied, completed, limited = manager.track(
            group_id="orders",
            data=order,
        )
        
        if dirtied:
            print(f"✅ Tracked: {order['order_id']} - ${order['amount']} ({order['customer_tier']})")
        else:
            print(f"❌ Rejected: {order['order_id']} - ${order['amount']} ({order['customer_tier']}, {order['region']})")
    
    # Show final count
    trackers = manager.get_trackers(group_id="orders", name="high_value_orders")
    print("\n" + "-" * 60)
    print(f"High-value orders tracked: {trackers[0].progress}/{trackers[0].target}")


if __name__ == "__main__":
    main()
```

## Summary

These examples demonstrate the flexibility of dictrack:

| Example | Use Case | Key Features |
|---------|----------|---------------|
| 1 | User Activity | Event handling, milestones |
| 2 | Rate Limiter | CountLimiter, reset policy |
| 3 | Order Processing | Multiple trackers, conditions |
| 4 | Campaign Analytics | Accumulation, multiple conditions |
| 5 | Session Monitoring | TimeLimiter, time-based tracking |
| 6 | Custom Condition | Custom business logic |

For more information, see:
- **[Getting Started](getting-started.md)**: Basic installation and concepts
- **[Advanced Usage](advanced-usage.md)**: Detailed feature explanations
- **[API Reference](api/manager.md)**: Complete API documentation

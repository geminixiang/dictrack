# Advanced Usage

This guide covers advanced features of dictrack including events, limiters, and custom configurations.

## Events System

 dictrack provides a powerful event system that allows you to react to tracking events.

### Event Types

| Event Code | Description |
|------------|-------------|
| `EVENT_TRACKER_ADDED` | Triggered when a tracker is added |
| `EVENT_TRACKER_STAGE_COMPLETED` | Triggered when a tracker reaches a stage |
| `EVENT_TRACKER_ALL_COMPLETED` | Triggered when all targets are completed |
| `EVENT_TRACKER_RESET` | Triggered when a tracker is reset |
| `EVENT_TRACKER_LIMITED` | Triggered when a tracker hits a limit |

### Listening to Events

```python
from dictrack.manager import TrackingManager
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.trackers.numerics.count import CountTracker
from dictrack.events import (
    EVENT_TRACKER_ADDED,
    EVENT_TRACKER_STAGE_COMPLETED,
    EVENT_TRACKER_ALL_COMPLETED,
    EVENT_TRACKER_LIMITED,
    TrackerEvent,
)

# Initialize manager
manager = TrackingManager(
    data_cache=MemoryDataCache(),
    data_store=MongoDBDataStore(host="localhost", database="dictrack"),
)

# Define event handler
def on_tracker_added(event):
    print(f"Tracker added: {event.name} to group {event.group_id}")

def on_tracker_completed(event):
    print(f"Tracker completed: {event.name}")

def on_tracker_limited(event):
    print(f"Tracker limited: {event.name}")

# Register listeners
manager.add_listener(EVENT_TRACKER_ADDED, on_tracker_added)
manager.add_listener(EVENT_TRACKER_ALL_COMPLETED, on_tracker_completed)
manager.add_listener(EVENT_TRACKER_LIMITED, on_tracker_limited)

# Now events will be triggered
tracker = CountTracker(name="MyTracker", target=5)
manager.add_tracker(group_id="group1", tracker=tracker)
```

## Limiters

Limiters control when a tracker should stop executing.

### CountLimiter

The `CountLimiter` limits operations by count:

```python
from dictrack.limiters.count import CountLimiter
from dictrack.trackers.numerics.count import CountTracker
from dictrack.manager import TrackingManager

manager = TrackingManager(
    data_cache=MemoryDataCache(),
    data_store=MongoDBDataStore(host="localhost", database="dictrack"),
)

# Create tracker with limiter
tracker = CountTracker(
    name="LimitedTracker",
    target=100,
    limiters=[CountLimiter(count=10)],  # Only 10 operations allowed
)

manager.add_tracker(group_id="group1", tracker=tracker)

# After 10 operations, the tracker will be limited
for i in range(15):
    is_ok, dirtied, completed, limited = manager.track(group_id="group1", data={"value": i})
    if limited:
        print(f"Tracker limited at iteration {i}")
        break
```

### TimeLimiter

The `TimeLimiter` limits operations within a time window:

```python
from dictrack.limiters.time import TimeLimiter
from dictrack.trackers.numerics.count import CountTracker
import time

# Create tracker with time limiter (60 seconds)
tracker = CountTracker(
    name="TimeLimitedTracker",
    target=1000,
    limiters=[TimeLimiter(seconds=60)],
)
```

### Custom Limiter

You can create custom limiters by extending `BaseLimiter`:

```python
from dictrack.limiters.base import BaseLimiter

class CustomLimiter(BaseLimiter):
    def __init__(self, max_value):
        super(CustomLimiter, self).__init__()
        self.max_value = max_value
        self.remaining = max_value

    def post_track(self, data, post_tracker, *args, **kwargs):
        self.remaining -= 1
        
        if self.remaining <= 0:
            self.limited = True
            return False
        return True

    def reset(self, *args, **kwargs):
        super(CustomLimiter, self).reset()
        self.remaining = kwargs.get("reset_value", self.max_value)
        return True
```

## Conditions

Conditions determine which data should be tracked.

### KeyValueEQ - Equal Condition

```python
from dictrack.conditions.keys import KeyValueEQ

# Track only when status equals "active"
tracker = CountTracker(
    name="ActiveTracker",
    target=10,
    conditions=[KeyValueEQ(key="status", value="active")],
)
```

### KeyValueNE - Not Equal Condition

```python
from dictrack.conditions.keys import KeyValueNE

# Track only when status is not "inactive"
tracker = CountTracker(
    name="ActiveOnlyTracker",
    target=10,
    conditions=[KeyValueNE(key="status", value="inactive")],
)
```

### Comparison Conditions

```python
from dictrack.conditions.keys import KeyValueGT, KeyValueGE, KeyValueLT, KeyValueLE

# Greater than
tracker = CountTracker(
    name="AdultTracker",
    target=10,
    conditions=[KeyValueGT(key="age", value=18)],
)

# Greater than or equal
tracker = CountTracker(
    name="SeniorTracker",
    target=10,
    conditions=[KeyValueGE(key="age", value=65)],
)
```

### KeyExists - Key Existence Check

```python
from dictrack.conditions.keys import KeyExists, KeyNotExists

# Track only if email field exists
tracker = CountTracker(
    name="EmailTracker",
    target=10,
    conditions=[KeyExists(key="email")],
)

# Track only if status field does NOT exist
tracker = CountTracker(
    name="NoStatusTracker",
    target=10,
    conditions=[KeyNotExists(key="status")],
)
```

### String Containment

```python
from dictrack.conditions.keys import KeyValueContained, KeyValueNotContained

# Track if status contains "active" (case insensitive by default)
tracker = CountTracker(
    name="ActiveContainsTracker",
    target=10,
    conditions=[KeyValueContained(key="status", value="active")],
)

# Case sensitive
tracker = CountTracker(
    name="ExactMatchTracker",
    target=10,
    conditions=[KeyValueContained(key="status", value="Active", case_sensitive=True)],
)
```

### List Conditions

```python
from dictrack.conditions.keys import KeyValueInList, KeyValueNotInList
from dictrack.conditions.keys import KeyValueListHasItem, KeyValueListNotHasItem
from dictrack.conditions.keys import KeyValueListIntersectList, KeyValueListNotIntersectList

# Track if role is in the allowed list
tracker = CountTracker(
    name="AllowedRoleTracker",
    target=10,
    conditions=[KeyValueInList(key="role", value=["admin", "moderator", "user"])],
)

# Track if tags contain specific item
tracker = CountTracker(
    name="HasTagTracker",
    target=10,
    conditions=[KeyValueListHasItem(key="tags", value="important")],
)

# Track if tags intersect with target list
tracker = CountTracker(
    name="TagIntersectionTracker",
    target=10,
    conditions=[KeyValueListIntersectList(key="tags", value=["urgent", "news", "update"])],
)
```

### Multiple Conditions

You can combine multiple conditions:

```python
from dictrack.conditions.keys import KeyValueEQ, KeyValueGT

# Track only when status is "active" AND age > 18
tracker = CountTracker(
    name="ComplexTracker",
    target=10,
    conditions=[
        KeyValueEQ(key="status", value="active"),
        KeyValueGT(key="age", value=18),
    ],
)
```

## Tracker Reset

You can reset trackers to their initial state:

```python
from dictrack.trackers import ResetPolicy

# Reset progress only
manager.reset_tracker(group_id="group1", name="MyTracker", reset_policy=ResetPolicy.PROGRESS)

# Reset limiter only
manager.reset_tracker(group_id="group1", name="MyTracker", reset_policy=ResetPolicy.LIMITER)

# Reset everything (progress + limiter)
manager.reset_tracker(group_id="group1", name="MyTracker", reset_policy=ResetPolicy.ALL)
```

## Multi-Target Tracking

Trackers can have multiple targets:

```python
from dictrack.trackers.numerics.count import CountTracker

tracker = CountTracker(
    name="MultiTargetTracker",
    target=[10, 20, 30],  # Multiple targets
)

# Track data
manager.track(group_id="group1", data={"status": "active"})

# Check current stage
print(tracker.progress)  # Current progress
print(tracker.target)    # Current target
```

## Persistence and Caching

### Expiration Settings

You can set expiration for trackers:

```python
# Expire after 3600 seconds (1 hour)
manager.add_tracker(
    group_id="group1",
    tracker=tracker,
    expire=3600,
)

# Expire at specific timestamp
import time
expire_at = int(time.time()) + 86400  # 24 hours from now
manager.add_tracker(
    group_id="group1",
    tracker=tracker,
    expire_at=expire_at,
)
```

### Manual Cache Management

```python
# Check if group is cached
if manager.data_cache.is_cached("group1"):
    # Get from cache
    trackers = manager.get_trackers(group_id="group1")

# Remove tracker
manager.remove_tracker(group_id="group1", name="MyTracker")

# Remove all trackers in group
manager.remove_tracker(group_id="group1")

# Flush all data (careful!)
manager.flush(confirm=True)
```

## Next Steps

- **[Examples](examples.md)**: Real-world usage examples
- **[API Reference](api/manager.md)**: Detailed API documentation

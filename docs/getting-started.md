# Getting Started

Welcome to the **dictrack** documentation! This guide will help you get started with installing and using dictrack in your project.

## What is dictrack?

`dictrack` is a powerful dictionary tracking tool designed for **condition-based monitoring and management**. It allows developers to easily track and handle dynamic data using flexible components such as conditions, targets, and limiters.

## Installation

### Basic Installation

Install the latest version of `dictrack` via pip:

```bash
pip install dictrack
```

### With Optional Dependencies

#### Redis Support (for caching)

```bash
pip install dictrack[redis]
```

#### MongoDB Support (for persistent storage)

```bash
pip install dictrack[mongodb]
```

#### Gevent Support (for asynchronous tasks)

```bash
pip install dictrack[gevent]
```

#### In-Memory Support (lightweight caching)

```bash
pip install dictrack[memory]
```

#### All Dependencies

```bash
pip install dictrack[redis,mongodb,gevent,memory]
```

## Quick Start

### Basic Example

Here's a simple example to get you started:

```python
from dictrack.conditions.keys import KeyValueEQ
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker

# Initialize the tracking manager
manager = TrackingManager(
    data_cache=MemoryDataCache(),
    data_store=MongoDBDataStore(host="localhost", port=27017, database="dictrack"),
)

# Create a tracker with a condition
tracker = CountTracker(
    name="MyTracker",
    target=10,  # Track 10 items
    conditions=[KeyValueEQ(key="status", value="active")],
)

# Add tracker to a group
manager.add_tracker(group_id="Robot001", tracker=tracker)

# Feed data into the tracker
data = {"id": 1, "status": "active"}
is_ok, dirtied, completed, limited = manager.track(group_id="Robot001", data=data)

# Check results
trackers = manager.get_trackers(group_id="Robot001", name="MyTracker")
print(trackers)
# Output: [<CountTracker (target=10 ... progress=1)>]
```

## Core Concepts

### 1. TrackingManager

The `TrackingManager` is the central component that manages all tracking operations.

```python
from dictrack.manager import TrackingManager
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore

manager = TrackingManager(
    data_cache=MemoryDataCache(),
    data_store=MongoDBDataStore(host="localhost", database="dictrack"),
)
```

### 2. Trackers

Trackers are the core objects that track progress towards a target. There are several types:

- **CountTracker**: Tracks the number of occurrences
- **NumericTracker**: Tracks numeric values
- **AccumulationTracker**: Accumulates values over time

```python
from dictrack.trackers.numerics.count import CountTracker
from dictrack.trackers.numerics.numeric import NumericTracker
from dictrack.trackers.numerics.accumulation import AccumulationTracker
```

### 3. Conditions

Conditions filter which data should be tracked. Common conditions include:

- `KeyExists`: Check if a key exists
- `KeyValueEQ`: Check if a key equals a value
- `KeyValueNE`: Check if a key is not equal to a value
- `KeyValueGT/GE/LT/LE`: Comparison conditions

```python
from dictrack.conditions.keys import KeyValueEQ, KeyExists, KeyValueGT

# Only track data where status == "active"
condition = KeyValueEQ(key="status", value="active")

# Only track data where age > 18
condition = KeyValueGT(key="age", value=18)
```

### 4. Limiters

Limiters control how many times a tracker can execute:

- **CountLimiter**: Limits by count
- **TimeLimiter**: Limits by time

```python
from dictrack.limiters.count import CountLimiter
from dictrack.limiters.time import TimeLimiter

# Allow only 5 operations
limiter = CountLimiter(count=5)

# Allow operations within 60 seconds
limiter = TimeLimiter(seconds=60)
```

### 5. Data Cache & Data Store

- **Data Cache**: Temporary storage (Redis, Memory)
- **Data Store**: Permanent storage (MongoDB)

```python
from dictrack.data_caches.redis import RedisDataCache
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.data_stores.mongodb import MongoDBDataStore
```

## Next Steps

- **[Advanced Usage](advanced-usage.md)**: Learn about events, limiters, and advanced configurations
- **[Examples](examples.md)**: Real-world usage examples
- **[API Reference](api/manager.md)**: Detailed API documentation

# dictrack

<p align="center">
  <a href="https://pypi.org/project/dictrack/"><img src="https://img.shields.io/pypi/pyversions/dictrack.svg?color=%2334D058" alt="Supported Python versions"></a>
  <a href="https://pypi.org/project/dictrack/"><img src="https://img.shields.io/pypi/v/dictrack.svg" alt="PyPI"></a>
  <a href="https://github.com/Bonislolz/dictrack/blob/main/LICENSE"><img src="https://img.shields.io/github/license/Bonislolz/dictrack.svg" alt="License"></a>
  <a href="https://pypi.ai/project/dictrack"><img src="https://static.pepy.tech/badge/dictrack/month" alt="Downloads"></a>
</p>

<!-- more -->

`dictrack` is a powerful **dictionary data tracking tool** designed for condition-based monitoring and management. It allows developers to easily track and handle dynamic data using flexible components such as conditions, targets, and limiters.

---

## Why use dictrack?

* **Multi-Target Support** — Track multiple targets simultaneously with customizable logic and stages.
* **Condition-Based Filtering** — Easily define conditions like key existence, value comparisons, or custom rules.
* **Flexible Limiting** — Use built-in limiters such as `CountLimiter` or `TimeLimiter` to control task execution.
* **Data Caching & Persistence** — Integrate `Redis` for caching and `MongoDB` for long-term storage.
* **Event System** — Includes events like `LIMITED`, `STAGE_COMPLETED`, and `ALL_COMPLETED` for fine-grained tracking.
* **Extensible Design** — Add custom conditions, targets, or limiters to adapt to unique requirements.
* **Python 2 & 3 Compatible** — Seamlessly supports both Python 2.7 and Python 3.7+ environments.

---

## Quick Start

Install dictrack:

```bash
pip install dictrack
```

Or with optional dependencies:

```bash
pip install dictrack[redis,mongodb]
```

---

## Example

Here's a simple example to get you started:

```python
from dictrack.conditions.keys import KeyValueEQ
from dictrack.data_caches.memory import MemoryDataCache
from dictrack.manager import TrackingManager
from dictrack.trackers.numerics.count import CountTracker

# Initialize tracker manager
manager = TrackingManager(
    data_cache=MemoryDataCache(),
    data_store=None,
)

# Add a tracker with conditions
tracker = CountTracker(
    name="Demo-1",
    conditions=[KeyValueEQ(key="status", value="active")],
    target=10,
)
group_id = "Robot001"
manager.add_tracker(group_id=group_id, tracker=tracker)

# Feed data into the manager
data = {"id": 1, "status": "active"}
manager.track(group_id=group_id, data=data)

# Check results
print(manager.get_trackers(group_id=group_id, name="Demo-1"))
# Output: [<CountTracker (target=10 conditions=set([<KeyValueEQ...>) progress=1)>]
```

---

## Use Cases

- **Condition-Based Monitoring** — Track data that meets specific conditions like key existence, value comparisons, or custom rules.
- **Rate Limiting** — Implement rate limiting for APIs or services using `CountLimiter` and `TimeLimiter`.
- **Multi-Stage Campaigns** — Track progress through multiple stages with multi-target support and milestone events.
- **IoT Device Monitoring** — Monitor device metrics and trigger alerts when thresholds are exceeded.
- **Session Analytics** — Track user session data including page views, purchases, and accumulated values.
- **Task Queue Management** — Manage and monitor task queues across multiple workers or robots.

---

## Related Projects

`dictrack` is commonly used with:

- **[Redis](https://redis.io/)** — In-memory data cache for fast access
- **[MongoDB](https://www.mongodb.com/)** — Persistent data storage
- **[FastAPI](https://fastapi.tiangolo.com/)** — Build APIs that integrate with dictrack

---

## Links

- [Getting Started](getting-started.md) — Quick start guide
- [Installation](installation.md) — Detailed installation instructions
- [Tutorial](tutorials/basic-tutorial.md) — Step-by-step tutorial
- [Advanced Usage](guides/advanced-usage.md) — Advanced features
- [Use Cases](guides/use-cases.md) — Real-world examples
- [API Reference](api/manager.md) — Complete API documentation
- [GitHub](https://github.com/Bonislolz/dictrack) — Source code and issues
+++
title = 'StepFunction 0.0.5 - An Update'
date = 2026-04-15T02:30:37+05:30
draft = false
+++

I have been using the `stepfunction` library across many of my projects where orchestration makes the code easier to reason about. While working with it, I kept running into a few missing pieces — smarter branching, built-in steps for common patterns like waiting and timeouts, and a way to handle transient failures gracefully. Version 0.0.5 addresses all of these.

## Callable Branch Routing

In earlier versions, branching worked by matching the step's return value against a fixed dictionary of keys. That meant your function had to return a specific string (like `"success"` or `"failure"`) for the router to know which step to go to next. This was limiting — especially when your function returns a rich object like a dict or a dataclass.

Now `branch` also accepts a callable. Instead of returning a routing key, you return whatever makes sense for your business logic, and a separate router function decides the next step. This keeps your step functions clean and your routing logic explicit.

```python
from asyncio import run
from stepfunction import StepFunction

def process_order(order: dict) -> dict:
    # Simulate order processing
    if order.get("item"):
        return {"status": "success", "total": 99.0}
    return {"status": "failed", "reason": "empty order"}

def route_order(result: dict) -> str:
    if result["status"] == "success":
        return "confirm_order"
    return "handle_failure"

def confirm_order(result: dict) -> dict:
    print(f"Order confirmed! Total: ${result['total']}")
    return result

def handle_failure(result: dict) -> dict:
    print(f"Order failed: {result['reason']}")
    return result

async def main():
    sf = StepFunction("OrderFlow")
    sf.add_step("process_order", process_order, branch=route_order)
    sf.add_step("confirm_order", confirm_order)
    sf.add_step("handle_failure", handle_failure)
    sf.set_start_step("process_order")

    await sf.execute(initial_input={"item": "Book"})

run(main())
```

The dict-based `branch` still works exactly as before — this is purely additive.

## WaitStep and TimeoutStep

Two patterns I found myself reimplementing constantly are pausing between steps and enforcing a time limit on a slow operation. Neither of these should require custom code every time.

- `WaitStep` pauses the workflow for a fixed number of seconds and passes the input through unchanged.
- `TimeoutStep` wraps any function — sync or async — and raises a `StepTimeoutError` if it runs longer than the allowed duration.

```python
from asyncio import run
from stepfunction import StepFunction
from stepfunction.steps import WaitStep, TimeoutStep

def fetch_data(payload: dict) -> dict:
    # Simulate a network call
    return {"data": "some result", **payload}

def process_data(data: dict) -> dict:
    return {**data, "processed": True}

async def main():
    sf = StepFunction("DataPipeline")
    sf.add_step("fetch", TimeoutStep(func=fetch_data, timeout=10.0), next_step="wait")
    sf.add_step("wait", WaitStep(duration=2), next_step="process")
    sf.add_step("process", process_data)
    sf.set_start_step("fetch")

    await sf.execute(initial_input={"query": "items"})
    print(sf.context)

run(main())
```

`TimeoutStep` works with both sync and async functions. Sync functions are offloaded to a thread pool so the timeout is enforced without blocking the event loop.

## RetryStep

APIs fail, services go down briefly, networks hiccup. `RetryStep` automatically tries the step again, waiting a bit between each attempt until it succeeds or runs out of retries.

```python
from asyncio import run
from stepfunction import StepFunction
from stepfunction.steps import RetryStep

attempt_count = 0

def flaky_api_call(payload: dict) -> dict:
    global attempt_count
    attempt_count += 1
    if attempt_count < 3:
        raise ConnectionError("Service temporarily unavailable")
    return {"data": "success", "attempts": attempt_count}

async def main():
    sf = StepFunction("APIFlow")
    sf.add_step(
        "fetch",
        RetryStep(func=flaky_api_call, max_retries=3, delay=1.0),
    )
    sf.set_start_step("fetch")

    await sf.execute(initial_input={"endpoint": "/data"})
    print(sf.last_result)  # {'data': 'success', 'attempts': 3}

run(main())
```

`max_retries=3` means the function can be called up to 4 times in total — the initial attempt plus three retries. `delay` is the number of seconds to wait between each attempt.

## Building Your Own Step

All built-in steps extend `BaseStep`. You can do the same to package reusable logic into a clean, configurable class:

```python
from typing import Any, Callable
from stepfunction.steps import BaseStep

class MultiplyStep(BaseStep):
    def __init__(self, factor: float):
        self.factor = factor

    def build(self) -> Callable:
        factor = self.factor

        async def run(input_value: Any) -> Any:
            return input_value * factor

        return run

# Usage
sf.add_step("double", MultiplyStep(factor=2), next_step="next_step")
```

Pass an instance directly to `add_step` — the framework calls `build()` for you.


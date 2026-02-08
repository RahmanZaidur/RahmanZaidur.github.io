---
title: "LangGraph + LangChain Runnables: The Runnable Smart Objects"
date: 2026-02-08
categories: [AI, LangChain, LangGraph]
tags: [langchain, langgraph, runnable, runnablelambda, python, streaming, batching, retries]
toc: true
---

If you're working with **LangGraph**, you'll quickly run into a core building block from **LangChain**: the **Runnable**.

A Runnable is the reason LangChain/LangGraph components compose so cleanly: you can **invoke**, **batch**, **stream**, **pipe**, run **in parallel**, and even add reliability patterns like **retries** and **fallbacks**—all through a consistent contract.

---

## What is a "Runnable"?

In standard Python, a function is just a block of code you call:

```python
my_function(input)
```

In LangChain, a **Runnable** is a **"Smart Object"** that **wraps** a function. Think of it like putting a standard engine (your function) inside a car chassis (the Runnable).

Why do we do this? A raw Python function can only do one thing: **run**.  
A Runnable object can do many things because it adheres to a strict contract (blueprint). It can:

- Run standard: `.invoke(input)`
- Run async: `.ainvoke(input)`
- Run in a batch: `.batch([input1, input2])` (often parallelized automatically)
- Run async batch: `.abatch([input1, input2])`
- Stream output: `.stream(input)` (get data chunk by chunk)
- Stream async: `.astream(input)`
- Chain: connect to other Runnables using the `|` symbol
- Run parallel branches: `RunnableParallel(...)` (fan-out)
- Add retries: `.with_retry(...)`
- Add fallback paths: `.with_fallbacks([...])`
- Pass runtime config: `config={...}` (tags/metadata/callbacks for tracing)
- Build dict-shaped flows: `.assign(...)` (add computed fields in a pipeline)

> Note: Some features depend on what the wrapped Runnable supports (for example, streaming typically requires a Runnable that yields chunks).

`RunnableLambda` is the specific wrapper that turns a plain Python function into this "Smart Object."

---

## Plain Python vs. Runnable

### Standard Python Function

```python
def add_one(x: int) -> int:
    return x + 1
```

Call it like normal Python:

```python
print(add_one(41))
```

Output:

```
42
```

### Runnable Version (RunnableLambda)

```python
from langchain_core.runnables import RunnableLambda

add_one_r = RunnableLambda(add_one)

print(add_one_r.invoke(41))
```

Output:

```
42
```

Now it supports the Runnable contract and can do much more than a raw function.

---

# Special stuff Runnables can do (with examples)

## Run standard: `.invoke(input)`

```python
from langchain_core.runnables import RunnableLambda

def area_of_circle(r: float) -> float:
    pi = 3.141592653589793
    return pi * r * r

r = RunnableLambda(area_of_circle)

print(r.invoke(2.0))
```

Output:

```
12.566370614359172
```

---

## Run async: `.ainvoke(input)`

If you're inside an async app (FastAPI, async worker, etc.), you can use the async methods.

```python
import asyncio
from langchain_core.runnables import RunnableLambda

def double(x: int) -> int:
    return 2 * x

r = RunnableLambda(double)

async def main():
    result = await r.ainvoke(21)
    print(result)

asyncio.run(main())
```

Output:

```
42
```

---

## Run in a batch: `.batch([input1, input2])`

Batching is great for throughput. Many Runnables can parallelize behind the scenes.

```python
from langchain_core.runnables import RunnableLambda

def square(x: int) -> int:
    return x * x

r = RunnableLambda(square)

print(r.batch([1, 2, 3, 4, 5]))
```

Output:

```
[1, 4, 9, 16, 25]
```

---

## Stream output: `.stream(input)`

Streaming is useful when you want incremental chunks of output.  
Not every Runnable streams, but when it does you can consume it as an iterator.

Here’s a simple streaming example that yields a running total (cumulative sum):

```python
from typing import Iterator, List
from langchain_core.runnables import RunnableLambda

def cumulative_sum(nums: List[int]) -> Iterator[int]:
    total = 0
    for n in nums:
        total += n
        yield total

r = RunnableLambda(cumulative_sum)

for chunk in r.stream([3, 1, 4, 1, 5]):
    print(chunk)
```

Output:

```
3
4
8
9
14
```

---

## Chain Runnables using `|`

One of the biggest benefits: **composition**.

```python
from langchain_core.runnables import RunnableLambda

def add_three(x: int) -> int:
    return x + 3

def times_ten(x: int) -> int:
    return x * 10

pipeline = RunnableLambda(add_three) | RunnableLambda(times_ten)

print(pipeline.invoke(4))
```

Output:

```
70
```

(Explanation: `(4 + 3) * 10 = 70`)

---

## Fan-out in parallel with `RunnableParallel`

Run multiple branches at once and get a dictionary back.

```python
from langchain_core.runnables import RunnableLambda, RunnableParallel

def square(x: int) -> int:
    return x * x

def cube(x: int) -> int:
    return x * x * x

parallel = RunnableParallel(
    squared=RunnableLambda(square),
    cubed=RunnableLambda(cube),
)

print(parallel.invoke(3))
```

Output:

```
{'squared': 9, 'cubed': 27}
```

---

## Build dict-shaped flows with `.assign(...)`

This is handy when your pipeline passes around dictionaries and you want to add computed fields.

```python
from langchain_core.runnables import RunnableLambda

def make_record(x: int) -> dict:
    return {"x": x}

def squared(d: dict) -> int:
    return d["x"] ** 2

def cubed(d: dict) -> int:
    return d["x"] ** 3

r = (
    RunnableLambda(make_record)
    .assign(square=RunnableLambda(squared))
    .assign(cube=RunnableLambda(cubed))
)

print(r.invoke(4))
```

Output:

```
{'x': 4, 'square': 16, 'cube': 64}
```

---

## Pass runtime config (tags/metadata)

Many LangChain components use config for tracing/observability.  
Even if your function ignores it, the Runnable interface supports it.

```python
from langchain_core.runnables import RunnableLambda

def add_tax(amount: float) -> float:
    tax_rate = 0.08
    return amount * (1 + tax_rate)

r = RunnableLambda(add_tax)

print(
    r.invoke(
        100.0,
        config={
            "tags": ["pricing", "tax"],
            "metadata": {"invoice_id": "INV-1001"},
        },
    )
)
```

Output:

```
108.0
```

---

## Reliability patterns: retry + fallback

### Retry with `.with_retry(...)`

```python
from langchain_core.runnables import RunnableLambda

attempts = {"n": 0}

def flaky_divide(x: int) -> float:
    attempts["n"] += 1
    if attempts["n"] < 3:
        raise RuntimeError("temporary failure")
    return 84 / x

r = RunnableLambda(flaky_divide).with_retry(stop_after_attempt=5)

print(r.invoke(2))
```

Output:

```
42.0
```

### Fallback with `.with_fallbacks([...])`

```python
from langchain_core.runnables import RunnableLambda

def primary(_: int) -> int:
    raise RuntimeError("primary failed")

def backup(x: int) -> int:
    return x * 2

r = RunnableLambda(primary).with_fallbacks([RunnableLambda(backup)])

print(r.invoke(21))
```

Output:

```
42
```

---

## Why this matters for LangGraph

LangGraph is about building **graphs of computation**. Under the hood:

- Node logic is often plain Python, frequently wrapped/used as Runnables
- A compiled LangGraph graph behaves like a **Runnable-style object** you can **invoke** and (often) **stream**

So once you understand the Runnable contract, LangGraph becomes much more intuitive:  
**graphs are composable computation blocks with multiple execution modes**.

---

## Quick reference (cheat sheet)

```python
r.invoke(x)                   # single input -> output
await r.ainvoke(x)            # async single input -> output

r.batch([x1, x2, x3])         # list of inputs -> list of outputs
await r.abatch([x1, x2, x3])  # async batch

r.stream(x)                   # yields chunks/events (if supported)
async for c in r.astream(x):  # async streaming (if supported)
    ...

r2 = r | other                # chain
r3 = r.with_retry(...)        # retry on failure
r4 = r.with_fallbacks([...])  # fallback paths

# dict-shaped pipelines:
r5 = r.assign(new_field=some_runnable)
```

---

## Wrap-up

A **Runnable** is a standardized, composable wrapper around computation. It unlocks a uniform way to run work:

- **once** (`invoke`)
- **in bulk** (`batch`)
- **incrementally** (`stream`)
- **as pipelines** (`|`)
- **in parallel** (`RunnableParallel`)
- **more robustly** (retry/fallback)
- **with runtime context** (config: tags/metadata)
- **with structured dict flows** (`assign`)

This contract is one of the key pieces that makes **LangChain** and **LangGraph** feel like LEGO bricks you can snap together.

---

---
title: "LangGraph + LangChain Runnables: The Runnable Smart Object"
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
- Run in parallel branches: `RunnableParallel(...)` (fan-out)
- Add retries: `.with_retry(...)`
- Add fallback paths: `.with_fallbacks([...])`
- Pass runtime config: `config={...}` (tags/metadata/callbacks for tracing)
- Assign or reshape fields in dict-like flows: `.assign(...)` (common in pipelines)
- Pick/route specific keys (when working with dict inputs): `RunnablePick(...)` / branching patterns

> Note: Some features depend on what the wrapped runnable supports (for example, streaming typically requires a runnable that yields chunks).

`RunnableLambda` is the specific wrapper that turns a plain Python function into this "Smart Object."

---

## Plain Python vs. Runnable

### Standard Python Function

```python
def add_exclamation(text: str) -> str:
    return text + "!"
```

Call it like normal Python:

```python
print(add_exclamation("hello"))
```

Text output:

```
hello!
```

### Runnable Version (RunnableLambda)

```python
from langchain_core.runnables import RunnableLambda

add_exclamation_r = RunnableLambda(add_exclamation)

print(add_exclamation_r.invoke("hello"))
```

Text output:

```
hello!
```

Now it supports the Runnable contract and can do much more than a raw function.

---

# Special stuff Runnables can do (with examples)

## Run standard: `.invoke(input)`

```python
from langchain_core.runnables import RunnableLambda

def normalize(s: str) -> str:
    return s.strip().lower()

r = RunnableLambda(normalize)

print(r.invoke("  HeLLo  "))
```

Text output:

```
hello
```

---

## Run async: `.ainvoke(input)`

If you're inside an async app (FastAPI, async worker, etc.), you can use the async methods.

```python
import asyncio
from langchain_core.runnables import RunnableLambda

def greet(name: str) -> str:
    return f"hi {name}"

r = RunnableLambda(greet)

async def main():
    result = await r.ainvoke("sam")
    print(result)

asyncio.run(main())
```

Text output:

```
hi sam
```

---

## Run in a batch: `.batch([input1, input2])`

Batching is great for throughput. Many Runnables can parallelize behind the scenes.

```python
from langchain_core.runnables import RunnableLambda

def square(x: int) -> int:
    return x * x

r = RunnableLambda(square)

print(r.batch([1, 2, 3, 4]))
```

Text output:

```
[1, 4, 9, 16]
```

---

## Stream output: `.stream(input)`

Streaming is useful when you want incremental chunks of output.  
Not every Runnable streams, but when it does you can consume it as an iterator.

Here’s a simple streaming example that yields one word at a time:

```python
from typing import Iterator
from langchain_core.runnables import RunnableLambda

def chunk_words(text: str) -> Iterator[str]:
    for w in text.split():
        yield w

r = RunnableLambda(chunk_words)

for chunk in r.stream("streaming feels fast"):
    print(chunk)
```

Text output:

```
streaming
feels
fast
```

---

## Chain Runnables using `|`

One of the biggest benefits: **composition**.

```python
from langchain_core.runnables import RunnableLambda

def strip_text(s: str) -> str:
    return s.strip()

def to_upper(s: str) -> str:
    return s.upper()

pipeline = RunnableLambda(strip_text) | RunnableLambda(to_upper)

print(pipeline.invoke("  hello  "))
```

Text output:

```
HELLO
```

---

## Fan-out in parallel with `RunnableParallel`

Run multiple branches at once and get a dictionary back.

```python
from langchain_core.runnables import RunnableLambda, RunnableParallel

def word_count(s: str) -> int:
    return len(s.split())

def char_count(s: str) -> int:
    return len(s)

parallel = RunnableParallel(
    words=RunnableLambda(word_count),
    chars=RunnableLambda(char_count),
)

print(parallel.invoke("count me"))
```

Text output:

```
{'words': 2, 'chars': 8}
```

---

## Add derived fields with `.assign(...)`

This is handy when your pipeline passes around dictionaries and you want to add computed fields.

```python
from langchain_core.runnables import RunnableLambda

def base(user_text: str) -> dict:
    return {"text": user_text}

def length(d: dict) -> int:
    return len(d["text"])

def uppercase(d: dict) -> str:
    return d["text"].upper()

r = (
    RunnableLambda(base)
    .assign(length=RunnableLambda(length))
    .assign(shout=RunnableLambda(uppercase))
)

print(r.invoke("hello"))
```

Text output:

```
{'text': 'hello', 'length': 5, 'shout': 'HELLO'}
```

---

## Pass runtime config (tags/metadata)

Many LangChain components use config for tracing/observability.  
Even if your function ignores it, the Runnable interface supports it.

```python
from langchain_core.runnables import RunnableLambda

def greet(name: str) -> str:
    return f"hi {name}"

r = RunnableLambda(greet)

print(
    r.invoke(
        "sam",
        config={
            "tags": ["demo", "greeting"],
            "metadata": {"request_id": "abc-123"},
        },
    )
)
```

Text output:

```
hi sam
```

---

## Reliability patterns: retry + fallback

### Retry with `.with_retry(...)`

```python
from langchain_core.runnables import RunnableLambda

attempts = {"n": 0}

def flaky(x: int) -> int:
    attempts["n"] += 1
    if attempts["n"] < 3:
        raise RuntimeError("temporary failure")
    return x * 10

r = RunnableLambda(flaky).with_retry(stop_after_attempt=5)

print(r.invoke(7))
```

Text output:

```
70
```

### Fallback with `.with_fallbacks([...])`

```python
from langchain_core.runnables import RunnableLambda

def primary(_: str) -> str:
    raise RuntimeError("primary is down")

def backup(_: str) -> str:
    return "backup result"

r = RunnableLambda(primary).with_fallbacks([RunnableLambda(backup)])

print(r.invoke("anything"))
```

Text output:

```
backup result
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

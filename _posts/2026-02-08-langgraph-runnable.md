---
title: "Understanding LangGraph Runnables: From Functions to Smart Objects"
date: 2026-02-08 12:00:00 +0000
categories: [AI, LangChain]
tags: [langgraph, python, runnable, tutorial]
math: true
---

If you are diving into LangChain or LangGraph, you will immediately encounter the concept of a **Runnable**. It is the fundamental building block of the entire ecosystem. But what exactly is it, and why should you care?

## 1. What is a "Runnable"?

In standard Python, a function is just a block of code you call: `my_function(input)`.

In LangChain, a **Runnable** is a "Smart Object" that *wraps* a function. Think of it like putting a standard engine (your function) inside a car chassis (the Runnable).

**Why do we do this?**

A raw Python function can only do one thing: run. A `Runnable` object can do many things because it adheres to a strict contract (blueprint). It can:

* **Run standard:** `.invoke(input)`
* **Run in a batch:** `.batch([input1, input2])` (Automatic parallel processing)
* **Stream output:** `.stream(input)` (Get data chunk by chunk)
* **Chain:** Connect to other Runnables using the `|` symbol.

`RunnableLambda` is the specific wrapper that turns a plain Python function into this "Smart Object."

---

## 2. The Transformation: Plain Python vs. Runnable

Let's look at a simple example to see the difference in syntax.

### Standard Python Function

```python
def square(x):
    return x * x

# Usage
print(square(5))
# Output: 25
```

### The Runnable Version

To convert this, we import `RunnableLambda` from the core LangChain package.

```python
from langchain_core.runnables import RunnableLambda

def square(x):
    return x * x

# Wrap it
runnable_square = RunnableLambda(square)

# Usage (Standard Execution)
print(runnable_square.invoke(5))
# Output: 25
```

At first glance, this looks like extra steps for the same result. But the magic happens when we use the **"Special Stuff"**—the capabilities that come with the Runnable contract.

---

## 3. The "Special Stuff": Runnable Capabilities

Once your function is wrapped, it gains superpowers that you don't have to write yourself.

### A. Automatic Batching (`.batch`)

In data science and AI, we often need to process lists of data. Usually, you would write a `for` loop or use a list comprehension. A Runnable handles this natively and can even optimize it using parallel processing (thread pools) automatically.

```python
# Processing a list of inputs simultaneously
results = runnable_square.batch([1, 2, 3, 4, 5])

print(results)
# Output: [1, 4, 9, 16, 25]
```

### B. Streaming (`.stream`)

If your function involves a Large Language Model (LLM) or a long calculation, you don't want to wait for the entire response before showing something to the user. Runnables allow you to stream the output chunk-by-chunk.

```python
import time

def slow_echo(text):
    for char in text:
        time.sleep(0.1) # Simulate delay
        yield char

echo_runnable = RunnableLambda(slow_echo)

# The output arrives character by character
for chunk in echo_runnable.stream("Hello World"):
    print(chunk, end="", flush=True)
```

### C. Chaining (`|`)

This is the most powerful feature. LangChain uses the Unix pipe syntax (`|`) to compose complex workflows. You can feed the output of one Runnable directly into the input of the next without writing glue code.

```python
def add_five(x):
    return x + 5

# Create two smart objects
runnable_square = RunnableLambda(square)
runnable_add = RunnableLambda(add_five)

# Create a chain: Square the number, THEN add five
chain = runnable_square | runnable_add

# (5 * 5) + 5 = 30
print(chain.invoke(5))
# Output: 30
```

## Summary

By wrapping your standard Python logic in a `Runnable`, you aren't just calling a function; you are creating a **deployable unit of logic** that fits perfectly into the LangGraph architecture. It unifies synchronous calls, batching, streaming, and composition under a single, consistent interface.

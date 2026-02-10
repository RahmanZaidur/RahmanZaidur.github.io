---
title: Mastering LangChain's ChatPromptTemplate
date: 2026-02-10
categories: [AI Engineering, LangChain]
tags: [llm, prompt engineering, ChatPromptTemplate, langchain, langgraph]
mermaid: true
toc: true
---

When building modern chat applications, we aren't just sending raw text to an LLM; we are managing a **conversation state**.

A chat model needs to distinguish between high-level instructions (System), current user input (Human), and the history of previous responses (AI). If you send everything as one big block of text, the model loses the context of "who said what," leading to hallucinations or ignored instructions.

This is where structured prompts become essential. **`ChatPromptTemplate`** is LangChain's solution for mapping raw input into the structured **list of messages** that LLMs require to maintain context, roles, and conversation history.

This tutorial covers the two primary methods for creating these templates: `.from_template` for simple inputs and `.from_messages` for complex, multi-turn conversations.

---

## Part 1: The Simple Method (`.from_template`)

The `.from_template()` method is the simplest entry point. It is used when your prompt is a standalone instruction or question without any prior conversation history or special system behavior.

It takes a single string and automatically converts it into a `HumanMessage`.

### Example: Single Input

In this example, we create a template that accepts a subject and generates a request about it.

```python
from langchain_core.prompts import ChatPromptTemplate

# Create a template from a simple string
# This automatically assumes the input is a "Human" message
template = ChatPromptTemplate.from_template("Tell me a fun fact about {subject}.")

# Invoke the template with data
messages = template.invoke({"subject": "octopuses"})

print(messages)
```

**Output:**
Note that the output is a list containing exactly one message with the type `HumanMessage`.

```text
messages=[HumanMessage(content='Tell me a fun fact about octopuses.')]
```

---

## Part 2: The Structured Method (`.from_messages`)

The `.from_messages()` method is the standard for building robust chat applications. It allows you to define a **sequence** of messages with specific roles.

This structure is required when you need to:
1.  **Set Behavior:** Give the AI a specific persona (System Message).
2.  **Provide Context:** Feed in previous conversation turns (History).
3.  **Teach by Example:** Show Few-Shot examples of Human/AI interaction.

### A. Using Role Tuples
The most concise way to define messages is using tuples in the format `("role", "content")`. The supported roles are:
* **system**: Sets the global behavior or persona (e.g., "You are a helpful assistant").
* **human**: Represents the user's input.
* **ai**: Represents a response from the model (often used for examples).

**Example:**
We will create a translator bot. We use a System message to enforce the behavior, and a Human/AI pair to show the model exactly how we want the output format to look.

```python
from langchain_core.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    # System: Defines behavior
    ("system", "You are a translator. Translate the user input into {language}."),
    
    # Few-shot Example (Human asking, AI answering)
    ("human", "Hello"),
    ("ai", "Hola"),
    
    # The actual user input
    ("human", "{text}")
])

messages = template.invoke({
    "language": "French", 
    "text": "Good morning"
})

print(messages)
```

**Output:**
The output is a structured list preserving the order and roles defined in the template.

```text
messages=[
    SystemMessage(content='You are a translator. Translate the user input into French.'), 
    HumanMessage(content='Hello'), 
    AIMessage(content='Hola'), 
    HumanMessage(content='Good morning')
]
```

---

### B. Using Message Classes
Instead of tuples, you can use specific Message classes (`SystemMessage`, `HumanMessage`, `AIMessage`). This offers more control if you are constructing messages dynamically in your code (e.g., pulling a system instruction from a database) or need to strictly type your inputs.

You can mix raw tuples and pre-made Message objects in the same list.

**Example:**
Here we use a `SystemMessage` object for the static instruction, but a tuple for the dynamic user input.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import SystemMessage, AIMessage, HumanMessage

# Pre-defined system instruction object
sys_msg = SystemMessage(content="You are a math tutor. Answer with numbers only.")

template = ChatPromptTemplate.from_messages([
    sys_msg,
    ("human", "What is 1 + 1?"),
    AIMessage(content="2"),
    ("human", "{math_problem}")
])

messages = template.invoke({"math_problem": "What is 5 * 5?"})

print(messages)
```

**Output:**

```text
messages=[
    SystemMessage(content='You are a math tutor. Answer with numbers only.'), 
    HumanMessage(content='What is 1 + 1?'), 
    AIMessage(content='2'), 
    HumanMessage(content='What is 5 * 5?')
]
```

---

### C. Using Placeholders (`MessagesPlaceholder`)
In a real conversation, the history is dynamic. You might have 2 previous messages, or 50. You cannot hardcode these slots in your template.

To solve this, you use a **placeholder**. This tells the template: *"I have a list of messages in a variable; please unpack them right here."*

You use the tuple syntax `("placeholder", "{variable_name}")` to designate this spot.

**Example:**
We will create a template that accepts a `conversation_history` list and injects it between the system instruction and the new user input.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import HumanMessage, AIMessage

template = ChatPromptTemplate.from_messages([
    ("system", "You are a helpful assistant."),
    
    # The Placeholder: Expects a list of messages in the variable 'history'
    ("placeholder", "{history}"),
    
    ("human", "{new_input}")
])

# Simulate a conversation history (e.g., from a database or memory)
existing_history = [
    HumanMessage(content="My name is Alice."),
    AIMessage(content="Hello Alice! How can I help you?")
]

# Invoke the template
messages = template.invoke({
    "history": existing_history,
    "new_input": "What is my name?"
})

print(messages)
```

**Output:**
Notice how the `existing_history` list was "flattened" into the main list. The placeholder was replaced by the two messages inside `existing_history`.

```text
messages=[
    SystemMessage(content='You are a helpful assistant.'), 
    HumanMessage(content='My name is Alice.'), 
    AIMessage(content='Hello Alice! How can I help you?'), 
    HumanMessage(content='What is my name?')
]
```

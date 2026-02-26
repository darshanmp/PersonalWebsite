---
layout: post
title: "Python Decorators: A Deep Dive"
date: 2026-01-10
categories: [Python, Programming]
---

Decorators are one of Python's most powerful and elegant features, yet they're often misunderstood or underutilized. After using them extensively in production codebases, I want to walk through how they really work — from the basics to composing multiple decorators.

## What is a Decorator?

At its core, a decorator is just a function that takes another function as an argument and returns a modified version of it. Python's `@` syntax is purely syntactic sugar.

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before the call")
        result = func(*args, **kwargs)
        print("After the call")
        return result
    return wrapper

@my_decorator
def greet(name):
    print(f"Hello, {name}!")
```

This is exactly equivalent to:

```python
greet = my_decorator(greet)
```

## Preserving Function Metadata

One gotcha beginners often hit: the wrapped function loses its original `__name__` and `__doc__` attributes. Fix this with `functools.wraps`:

```python
import functools

def my_decorator(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

Always use `@functools.wraps` — it's a best practice that makes debugging and introspection much cleaner.

## Decorators with Arguments

Sometimes you need to pass arguments to the decorator itself. This requires an extra layer of nesting:

```python
def repeat(times):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def say_hello():
    print("Hello!")
```

## Class-Based Decorators

For stateful decorators, using a class can be cleaner:

```python
class Cache:
    def __init__(self, func):
        functools.update_wrapper(self, func)
        self.func = func
        self._cache = {}

    def __call__(self, *args):
        if args not in self._cache:
            self._cache[args] = self.func(*args)
        return self._cache[args]

@Cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

## Real-World Use Cases

Decorators shine in cross-cutting concerns that shouldn't pollute your business logic:

- **Logging** — automatically log function entry/exit and arguments
- **Authentication** — guard routes in Flask/Django with `@login_required`
- **Rate limiting** — throttle API calls
- **Caching** — Python's built-in `@functools.lru_cache` is a perfect example
- **Retry logic** — transparently retry flaky network calls

## Stacking Decorators

You can stack multiple decorators, but order matters. They apply bottom-up:

```python
@decorator_a
@decorator_b
def my_func():
    pass

# Equivalent to: my_func = decorator_a(decorator_b(my_func))
```

## Key Takeaways

- Decorators are functions that return functions — nothing magical
- Always use `@functools.wraps` to preserve metadata
- Use class-based decorators when you need state
- Stacking order is bottom-up

Once you internalize these patterns, you'll find yourself reaching for decorators naturally to keep your code DRY and concerns well-separated.

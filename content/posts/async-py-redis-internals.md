---
draft: false
title: "Python Async Queues & Redis Pub/Sub"
date: "2022-08-17T09:58:57+10:00"
tags: ["async", "pub/sub", "python", "queue", "redis", "slots"]
---

## Async Queues (`asyncio.Queue`)

Python's async queue is implemented as a FIFO queue. The two operations to be aware of, are:

- `coroutine get()` - waits until there is an item in the queue to then remove and return that item.
- `coroutine put()` - puts an item in the queue.

Each of the above also have their corresponding `get_nowait` and `put_nowait` methods to perform the same operation without blocking.

This allows for a producer / consumer model to be implemented. The below code will run both the consumer and producer at the same time, allowing the program to run a consumer as a "background" task while performing some other action.

```python
import asyncio


async def consumer(q: asyncio.Queue):
    while True:
        msg = await q.get()
        print("Retrieved message:", msg)


async def producer(q: asyncio.Queue):
    for i in range(10):
        await q.put("Sending message number:", i+1)
        await asyncio.sleep(0.5)


async def main():
    q = asyncio.Queue()
    await asyncio.gather(consumer(q), producer(q))


asyncio.run(main())
```

This is the base of how `aioredis` works.

## Async Redis (`aioredis`) - Pub/Sub

Subscribing to a channel in Redis will return `aioredis.Channel` object that is a wrapper around `asyncio.Queue`.

```python
import aioredis
import asyncio

CHANNEL = "chat"


async def consumer(c: aioredis.Channel):
    while True:
        msg = await c.get()
        print("Message:", msg)


async def producer(r: aioredis.Redis):
    for i in range(10):
        await r.publish(CHANNEL, f"Sending message number: {i+1}")
        await asyncio.sleep(0.5)


async def main():
    redis = await aioredis.create_redis_pool(("localhost", 6379))
    ch, *_ = await redis.subscribe(CHANNEL)

    await asyncio.gather(consumer(ch), producer(redis))

    redis.close()


asyncio.run(main())
```

Additionally, this of course works with manually publishing via the `redis-cli`:

```bash
>>> docker run --rm -p 6379:6379 -d redis:7.0.2-alpine
56017740cc4911a080a35373e0de50b12b32c2a28f56ecea92a33eb1efd9a79a
>>> docker exec -it 56 sh
/data # redis-cli
127.0.0.1:6379> PUBLISH chat hi
```

---

## Python Slots

This might be a bit random to add to this post, however it was found that a few of the classes in Django Channels / Async Redis were using slots.

So let's first start off with some basic Python code demonstrating a class with some attributes.

```python
>>> class Foo:
>>>     def __init__(self, a, b):
>>>         self.a = a
>>>         self.b = b
>>>
>>> foo = Foo(a=3, b=5)
>>> foo.c = 8
>>> foo.__dict__
{'a': 3, 'b': 5, 'c': 8}
```

Slots is a feature for Python classes, where by the `__slots__` class variable is assigned a list of strings that are variable names. This is used by Python to reserve memory, and prevent the creation of the `__dict__` attribute. It also prevents the creation of variables that aren't declared in `__slots__`. In other words, it's more efficient in terms of memory space and speed of access when using slots.

Here's an example of what it looks like.

```python
>>> class Foo:
>>>     __slots__ = ("a", "b")
>>>
>>>     def __init__(self, a, b):
>>>         self.a = a
>>>         self.b = b
>>>
>>> foo = Foo(a=3, b=5)
>>> foo.c = 8
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Foo' object has no attribute 'c'
>>> foo.__dict__
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'Foo' object has no attribute '__dict__'
```

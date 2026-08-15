---
tags:

- python
- tips
- advanced
- programming

---

# Python Tips & Tricks

A collection of Python code tips and tricks that will be useful for projects and work-related development.

## Name Mangling

This is a very confusing feature that is important to understand to avoid mistakes in the future. I learned from [this YouTube video](https://youtu.be/0hrEaA3N3lk) and here's what's important to know.

When you create a double underscore variable inside a class, at compilation time, Python changes its name to be `_{ClassName}__{VariableName}`. For example:

```python
class A:
    __count = 0  # Python will change internally the name to _A__count

    def get_count(self):
        return __count
```

This might cause confusion because when trying to access the variable from outside, like `A.__count`, you'll not be accessing the same variable defined inside the class.

Nevertheless, it's a useful feature because if you define a variable with the same name as one in the parent class, the values will not be mixed up, removing the need to know the implementation details of the parent class. However, it might cause some confusion. Use it carefully.

## Variable Scopes and Closures

Variable scopes define where to look for a variable in Python. Closures are the environment where functions store data about their values. More details in [this video](https://youtu.be/jXugs4B3lwU).

The key principle is:

> Lookup happens at runtime, location is decided at compile time.

What does this mean? When you define a function and use a variable, Python decides whether to look at local or global scopes at compile time, but the actual value lookup happens at runtime. For example:

```python
x = 10

def function():
    print(x)
    x = 20  # This makes x local to the function

function()  # Gives UnboundLocalError
```

The above code will return an error because `x` is found to be local at compile time, but at runtime, `x` is accessed before it's assigned locally.

## Concurrency and Parallelism

Understanding the difference between concurrency and parallelism is crucial for writing efficient Python programs.

- **Concurrency** means dealing with multiple tasks that are *in progress* at the same
  time — the CPU (or a single core) switches between them, making progress on each without
  necessarily running any two at the exact same instant. It's about structure: how you
  organize work that can be interleaved.
- **Parallelism** means multiple tasks *actually run at the same time*, on different CPU
  cores. It's about execution: doing more than one thing at the exact same moment.

A useful mental model: concurrency is one cook juggling several pots on the stove;
parallelism is several cooks, each with their own stove, working at once. You can have
concurrency without parallelism (one core, many interleaved tasks), and you generally need
concurrency-friendly code to take advantage of parallelism.

See also: [Concurrency concepts (video)](https://youtu.be/GpqAQxH1Afc).

### The Global Interpreter Lock (GIL)

CPython (the standard Python implementation) has a **Global Interpreter Lock**: only one
thread can execute Python bytecode at a time, even on a multi-core machine. This means
plain Python threads don't give you true parallelism for CPU-bound work — two threads
crunching numbers won't run any faster than one, because they're taking turns holding the
GIL.

The GIL is released, however, during I/O waits (network calls, file reads, `time.sleep`,
etc.) and inside many C-extension calls (like NumPy's heavy lifting). That's why threading
still pays off for I/O-bound work.

### Threading

Threading in Python allows multiple threads to run within the same process, sharing memory
space. Because of the GIL, threading is a good fit for **I/O-bound** work — tasks that
spend most of their time waiting (HTTP requests, database queries, disk I/O) — since one
thread can run while another is blocked waiting on I/O:

```python
import threading
import time


def download(name: str) -> None:
    print(f"Starting {name}")
    time.sleep(2)  # simulates a network call
    print(f"Finished {name}")


threads = [threading.Thread(target=download, args=(f"file-{i}",)) for i in range(3)]
for t in threads:
    t.start()
for t in threads:
    t.join()
```

The three "downloads" above overlap in wall-clock time even though only one thread ever
executes Python bytecode at a given instant, because they spend most of their time
waiting, not computing.

### Multiprocessing

For **CPU-bound** work (heavy computation, data crunching, image processing), threading
won't help because of the GIL. `multiprocessing` sidesteps this by spawning separate OS
processes, each with its own Python interpreter and its own GIL, giving you real
parallelism across CPU cores:

```python
from multiprocessing import Pool


def square(n: int) -> int:
    return n * n


if __name__ == "__main__":
    with Pool(processes=4) as pool:
        results = pool.map(square, range(10))
    print(results)
```

The trade-off is overhead: spawning processes and moving data between them (via pickling)
is more expensive than spawning threads, so multiprocessing pays off for larger, coarser
chunks of work rather than many tiny tasks.

### Rule of Thumb

| Workload type | Bottleneck | Use |
|---|---|---|
| I/O-bound (network, disk, DB) | Waiting on external systems | `threading` or `asyncio` |
| CPU-bound (computation) | The GIL | `multiprocessing` |

Learn more: [Multiprocessing in Python (video)](https://youtu.be/X7vBbelRXn0).

## Related Articles

- [Flask Framework](../web-development/flask-framework.md)

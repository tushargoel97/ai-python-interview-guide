# Python Core

## 📑 Table of Contents

1. [1. Python Context Managers (`with` statement)](#1-python-context-managers-with-statement)
2. [2. Python Decorators](#2-python-decorators)
3. [3. Python Multithreading vs Multiprocessing](#3-python-multithreading-vs-multiprocessing)
4. [4. Python OOP — `super()`, Private/Protected, Key Concepts](#4-python-oop--super-privateprotected-key-concepts)
5. [5. GIL (Global Interpreter Lock) & Garbage Collector](#5-gil-global-interpreter-lock--garbage-collector)
6. [6. Iterators & Generators](#6-iterators--generators)
7. [7. Concurrency vs Parallelism](#7-concurrency-vs-parallelism)
8. [8. Async in Python & FastAPI](#8-async-in-python--fastapi)
9. [9. Python Concepts Deep Dive](#9-python-concepts-deep-dive)
10. [10. Python Type Hints & `pydantic`](#10-python-type-hints--pydantic)

## 1. Python Context Managers (`with` statement)

> **📣 Interview-ready definition:** _"A context manager is a Python object that automatically handles setup and cleanup around a block of code — typically using the `with` statement. It guarantees that resources (files, connections, locks) get properly released even if an error occurs in between. The two methods that make it work are `__enter__` (called at the start) and `__exit__` (called at the end, even on exception)."_

### 🧑‍🍳 Layman Explanation

A context manager is like a **responsible roommate** who always turns off the lights when leaving a room. You walk in (enter the context), do your thing, and no matter what happens — even if you trip and fall — the roommate makes sure the lights are turned off (cleanup happens). You never forget to close a file, release a lock, or close a DB connection.

Real-life analogy: Checking into and out of a hotel. `__enter__` = check-in (you get a room key). `__exit__` = check-out (room is cleaned, key returned) — this happens automatically even if your trip is cut short by an emergency.

### 🔧 Technical Explanation

**The protocol:** Any object implementing `__enter__()` and `__exit__()` is a context manager.

```python
# Class-based context manager
class DatabaseConnection:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.conn = None

    def __enter__(self):
        self.conn = psycopg2.connect(self.connection_string)
        return self.conn  # this is what gets assigned to the `as` variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        # ALWAYS runs — even if an exception occurred inside the `with` block
        if self.conn:
            if exc_type:
                self.conn.rollback()   # exception happened → rollback
            else:
                self.conn.commit()     # no exception → commit
            self.conn.close()
        return False  # False = re-raise exception, True = suppress it

# Usage
with DatabaseConnection("postgres://...") as conn:
    cursor = conn.cursor()
    cursor.execute("INSERT INTO users ...")
    # if exception → rollback + close
    # if success → commit + close
```

**`contextlib` — the easier way:**

```python
from contextlib import contextmanager

@contextmanager
def timer(label):
    start = time.time()
    yield                    # everything before yield = __enter__, after = __exit__
    print(f"{label}: {time.time() - start:.2f}s")

with timer("DB query"):
    run_expensive_query()

# Async context manager
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_db_session():
    session = await create_session()
    try:
        yield session
    finally:
        await session.close()
```

**`__exit__` parameters explained:**

- `exc_type`: Exception class (e.g., `ValueError`) or `None` if no error
- `exc_val`: The exception instance
- `exc_tb`: Traceback object
- Return `True` to **suppress** the exception (swallow it). Return `False` (default) to let it propagate.

**Common built-in context managers:**

- `open("file.txt")` — auto-closes file
- `threading.Lock()` — auto-releases lock
- `decimal.localcontext()` — temporarily changes decimal precision
- `unittest.mock.patch()` — auto-restores original after test
- `tempfile.TemporaryDirectory()` — auto-deletes temp dir

**Context Manager vs Decorator — when they overlap:**

```python
# This acts as BOTH a decorator AND a context manager
@contextmanager
def suppress_errors():
    try:
        yield
    except Exception as e:
        logging.error(f"Suppressed: {e}")

# As context manager
with suppress_errors():
    risky_operation()

# As decorator (works because @contextmanager supports it)
@suppress_errors()
def risky_function():
    ...
```

### 📌 TLDR for Interviews

> "Context managers guarantee cleanup using `__enter__`/`__exit__` — the `with` statement ensures resources (files, connections, locks) are properly released even if exceptions occur. Use `contextlib.contextmanager` for the decorator shortcut. `__exit__` receives exception info and can suppress errors by returning `True`. They're essential for resource management, transaction handling, and temporary state changes."

---

## 2. Python Decorators

> **📣 Interview-ready definition:** _"A decorator is a function that takes another function as input and returns a new function that adds some behavior — like logging, timing, or authentication — without changing the original function's code. The `@decorator_name` syntax is just shorthand for `func = decorator_name(func)`."_

### 🧑‍🍳 Layman Explanation

Think of a decorator like a **gift wrapper**. You have a gift (your function), and you wrap it in fancy paper (the decorator) — the gift inside doesn't change, but the whole package looks/does something extra. A coffee shop might "decorate" every drink order by automatically adding a receipt and a loyalty stamp — you don't change how each drink is made, you just wrap the process.

### 🔧 Technical Explanation

A decorator is a **higher-order function** that takes a function as input, adds some behavior, and returns a new function — without modifying the original function's source code.

```python
import functools
import time

def timer(func):
    @functools.wraps(func)  # preserves original function metadata
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

@timer  # syntactic sugar for: fetch_data = timer(fetch_data)
def fetch_data(url):
    # expensive operation
    pass
```

**Key details interviewers love:**

- `@functools.wraps(func)` — preserves `__name__`, `__doc__`, `__module__` of the original function. Without it, debugging tools see "wrapper" instead of "fetch_data".
- Decorators can be **stacked**: `@auth @timer def view()` applies bottom-up (timer first, then auth).
- **Decorator with arguments** requires a triple-nested function (decorator factory → decorator → wrapper).
- Common real-world uses: `@login_required`, `@app.route()`, `@property`, `@staticmethod`, `@retry`.
- **Class-based decorators** use `__call__` to make instances callable.

### 📌 TLDR for Interviews

> "A decorator is a function that wraps another function to add behavior — like logging, auth, or caching — without modifying its source code. It's Python's implementation of the Decorator design pattern. Always use `@functools.wraps` to preserve the original function's metadata."

---

## 3. Python Multithreading vs Multiprocessing

> **📣 Interview-ready definition:** _"Multithreading runs multiple threads inside the same process — they share memory, but in CPython they can't truly run in parallel due to the GIL, so they only help for I/O-bound work like network calls. Multiprocessing runs separate processes, each with its own Python interpreter and memory — bypassing the GIL completely, so it gives true parallelism for CPU-bound work like data crunching."_

### 🧑‍🍳 Layman Explanation

**Multithreading** = One chef with two hands doing two tasks in the same kitchen. They share the same counter, fridge, and tools. They can switch between chopping and stirring quickly, but can only really do one thing at a time (because of a tiny kitchen rule — the GIL).

**Multiprocessing** = Two separate chefs in two separate kitchens. Each has their own fridge, counter, and tools. They truly work in parallel but can't easily share ingredients without passing them through a window.

### 🔧 Technical Explanation

| Aspect        | Multithreading                                  | Multiprocessing                        |
| ------------- | ----------------------------------------------- | -------------------------------------- |
| Memory        | Shared memory space                             | Separate memory per process            |
| GIL impact    | Limited by GIL for CPU-bound work               | Bypasses GIL completely                |
| Best for      | I/O-bound (network calls, file I/O, DB queries) | CPU-bound (image processing, ML, math) |
| Overhead      | Low (lightweight threads)                       | High (full process fork)               |
| Communication | Direct (shared variables, locks)                | IPC: Queue, Pipe, shared memory        |
| Risk          | Race conditions, deadlocks                      | Higher memory usage                    |

```python
# Multithreading — good for I/O-bound
from concurrent.futures import ThreadPoolExecutor
import requests

def fetch(url):
    return requests.get(url).status_code

with ThreadPoolExecutor(max_workers=10) as pool:
    results = pool.map(fetch, urls)  # 10 threads, each waiting on network

# Multiprocessing — good for CPU-bound
from concurrent.futures import ProcessPoolExecutor

def crunch(n):
    return sum(i*i for i in range(n))

with ProcessPoolExecutor(max_workers=4) as pool:
    results = pool.map(crunch, [10**7]*4)  # 4 separate processes
```

### 📌 TLDR for Interviews

> "Use **threading** for I/O-bound tasks (API calls, DB queries) where threads spend time waiting. Use **multiprocessing** for CPU-bound tasks (data crunching, image processing) because each process has its own GIL — enabling true parallelism across CPU cores."

---

## 4. Python OOP — `super()`, Private/Protected, Key Concepts

> **📣 Interview-ready definition:** _"Python's OOP is built on classes that bundle data and behavior, with inheritance enabling code reuse via parent-child relationships. `super()` delegates calls to a parent class following the MRO (Method Resolution Order). Python uses naming conventions for access control rather than enforcement — `_protected` is 'please don't touch', `__private` triggers name-mangling. Python trusts developers; it doesn't lock things down like Java."_

### 🧑‍🍳 Layman Explanation

OOP is like a **family tree**. A `Vehicle` is the grandparent. `Car` inherits from Vehicle (it IS a vehicle, with extra features). `ElectricCar` inherits from Car. When an ElectricCar needs to do something its parent already knows, it calls `super()` — like calling your parent for a recipe they already know, instead of reinventing it.

**Private/Protected** = A diary with a lock (`__private`) vs a diary that says "please don't read" (`_protected`). Python trusts you as an adult — the underscore is a _convention_, not a wall.

### 🔧 Technical Explanation

```python
class Animal:
    def __init__(self, name):
        self.name = name            # public
        self._species = "Unknown"   # protected (convention: internal use)
        self.__id = id(self)        # private (name-mangled to _Animal__id)

    def speak(self):
        return "..."

class Dog(Animal):
    def __init__(self, name, breed):
        super().__init__(name)  # calls Animal.__init__ — delegates to parent
        self.breed = breed

    def speak(self):            # method overriding (polymorphism)
        return "Woof!"
```

**Key OOP concepts to nail:**

| Concept           | What it means                                                                                                                                  |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `super()`         | Delegates a method call to a parent class. Follows the **MRO** (Method Resolution Order) — C3 linearization. Critical in multiple inheritance. |
| `_protected`      | Single underscore = convention: "internal, use at your own risk." Still fully accessible.                                                      |
| `__private`       | Double underscore = **name mangling**. `__x` in class `Foo` becomes `_Foo__x`. Not truly private — just harder to access accidentally.         |
| `@property`       | Turns a method into a getter; allows controlled access without changing the calling API.                                                       |
| `@classmethod`    | Receives `cls` as first arg. Used for alternative constructors (e.g., `from_json`).                                                            |
| `@staticmethod`   | No `self` or `cls`. A utility function that lives in the class namespace.                                                                      |
| `@abstractmethod` | From `abc` module. Forces subclasses to implement a method. Makes a class abstract (can't be instantiated).                                    |
| MRO               | `ClassName.__mro__` — the order Python searches for methods. Follows C3 linearization.                                                         |

```python
# Multiple inheritance + super() with MRO
class A:
    def greet(self): return "A"

class B(A):
    def greet(self):
        return "B -> " + super().greet()

class C(A):
    def greet(self):
        return "C -> " + super().greet()

class D(B, C):
    def greet(self):
        return "D -> " + super().greet()

print(D().greet())  # D -> B -> C -> A
print(D.__mro__)    # D, B, C, A, object
```

### 📌 TLDR for Interviews

> "`super()` delegates method calls to the parent using MRO — essential in multiple inheritance. Python uses naming conventions for access control: `_protected` is 'please don't touch', `__private` triggers name-mangling to `_ClassName__attr`. Python favors convention over enforcement — it trusts developers."

---

## 5. GIL (Global Interpreter Lock) & Garbage Collector

> **📣 Interview-ready definition:** _"The GIL is a mutex inside CPython that allows only one thread to execute Python bytecode at a time — it exists to protect reference counting from race conditions. It's why multithreading in Python doesn't give true parallelism for CPU-bound work, but still helps for I/O-bound work because the GIL is released during I/O. Garbage collection in Python is primarily reference counting (immediate cleanup when refcount hits zero), with a generational cyclic collector to handle reference cycles that simple counting can't."_

### 🧑‍🍳 Layman Explanation

**GIL** = Imagine a single microphone at a conference. Even if 10 speakers (threads) are ready, only one can talk at a time. They pass the mic around quickly, giving the illusion of simultaneous speech — but only one voice is ever active. That's CPython's GIL.

**Garbage Collector** = A janitor who counts how many people are using each chair. When nobody is using a chair (reference count = 0), the janitor removes it immediately. For tricky situations where two chairs reference each other but nobody actually sits in them (circular references), the janitor does a periodic deep clean (cycle detection).

### 🔧 Technical Explanation

---

#### What IS the GIL exactly?

The GIL (Global Interpreter Lock) is a **boolean mutex (mutual exclusion lock)** inside the CPython interpreter (the default Python implementation written in C). It is a single lock that protects access to **Python objects and the interpreter's internal state**.

At the C level, it's essentially:

```c
// Simplified — inside CPython's ceval.c
static PyThread_type_lock interpreter_lock;  // THIS is the GIL

// Before executing ANY bytecode instruction:
PyThread_acquire_lock(interpreter_lock);
// execute one or more bytecode instructions
PyThread_release_lock(interpreter_lock);
```

Every Python thread must **acquire the GIL** before it can execute any Python bytecode. Only one thread holds it at any given moment.

---

#### WHY does the GIL exist? (Significance)

The root cause: **CPython's memory management uses reference counting**, and reference counting is NOT thread-safe.

```python
# Every Python object in C has this:
# typedef struct {
#     Py_ssize_t ob_refcnt;   ← this counter
#     PyTypeObject *ob_type;
# } PyObject;

a = []      # ob_refcnt = 1
b = a       # ob_refcnt = 2 (two variables point to the same list)
del b       # ob_refcnt = 1
del a       # ob_refcnt = 0 → immediately freed
```

Without the GIL, if two threads simultaneously modified `ob_refcnt` of the same object, you'd get **race conditions** — the counter could end up wrong, leading to:

- **Memory leaks** (refcount too high → object never freed)
- **Use-after-free crashes** (refcount too low → object freed while still in use → segfault)

The GIL was chosen as the simplest solution: one global lock to protect ALL objects at once. The alternative — per-object fine-grained locks — would add massive overhead and complexity.

**Significance in practice:**

1. **CPU-bound threading is useless in CPython**: Multiple threads doing pure Python computation will NOT use multiple cores. They'll run sequentially, trading the GIL back and forth.
2. **I/O-bound threading still works**: The GIL is _released_ when a thread does I/O (network, disk, sleep). Other threads can run while one waits on I/O.
3. **C extensions can release the GIL**: NumPy, PIL, database drivers release the GIL during heavy C computations — so those CAN use multiple cores even with threading.
4. **It's why `multiprocessing` exists**: Each process has its own interpreter → its own GIL → true parallelism.

---

#### WHO controls the GIL? (How it switches between threads)

The GIL is controlled by **CPython's bytecode evaluation loop** (`ceval.c`). Here's how:

**Python 3.2+ (current mechanism — "new GIL"):**

- A thread holds the GIL and executes bytecode.
- Every **5 milliseconds** (configurable via `sys.setswitchinterval()`), the interpreter sets a flag: `eval_breaker`.
- When the running thread checks this flag (at safe points between bytecodes), it **voluntarily releases the GIL**.
- Waiting threads compete to acquire it. The OS thread scheduler decides who gets it next.
- This is a **cooperative + preemptive** hybrid: threads cooperate by checking the flag, but the timeout forces switching.

```python
import sys
print(sys.getswitchinterval())   # default: 0.005 (5ms)
sys.setswitchinterval(0.001)     # change to 1ms (more responsive, more overhead)
```

**When the GIL is released automatically:**

1. **I/O operations**: `socket.recv()`, `file.read()`, `time.sleep()` — the thread releases the GIL before the blocking call and reacquires after.
2. **C extension calls**: Libraries like NumPy wrap their C code with `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS` macros to release the GIL during computation.
3. **`await` in asyncio**: Not exactly GIL-related (single-threaded), but achieves similar cooperative yielding at the Python level.

```c
// Inside a C extension (e.g., NumPy matrix multiply):
Py_BEGIN_ALLOW_THREADS    // releases the GIL
result = heavy_c_computation(matrix_a, matrix_b);
Py_END_ALLOW_THREADS      // reacquires the GIL
// While this runs, OTHER Python threads can execute!
```

**Old GIL (Python < 3.2):**

- Used an instruction counter: switch every 100 bytecode instructions (`sys.setcheckinterval()`).
- Problem: bytecode instructions vary wildly in duration. A single `list.sort()` call could hog the GIL for seconds.
- The new time-based GIL (introduced by Antoine Pitrou) fixed this.

---

#### The Future: Free-threaded Python (no-GIL)

**Python 3.13 (Oct 2024 — PEP 703):**

- First release with experimental free-threaded build: `python3.13t` (the `t` suffix).
- Compile CPython with `--disable-gil`.
- Replaces global lock with **per-object locks + biased locking + deferred reference counting**.
- Allows true multi-threaded parallelism for CPU-bound Python code.
- Initial single-threaded overhead: ~10-40% (expected to improve each release).

**Python 3.14 (Oct 2025):**

- Free-threaded mode promoted from "experimental" to **supported (beta)**. Major C extensions (NumPy, Pydantic v2, uvloop) added `t`-build support.
- Single-threaded overhead reduced to ~5-10% through biased reference counting optimizations.
- Still a separate build (`python3.14t`); mainstream `python3.14` ships with the GIL.

**Python 3.15+ (2026-2027 roadmap):**

- Steering council targets making free-threading the **default** — the regular `python` binary would ship without the GIL.
- Legacy C extensions that haven't updated would need a GIL-enabled fallback build.
- Goal: full GIL removal once ecosystem readiness crosses a critical threshold.

**Note**: The GIL is a CPython implementation detail. Other Python implementations don't have it:

- **Jython** (Python on JVM) — no GIL, uses JVM threading.
- **IronPython** (.NET) — no GIL.
- **PyPy** — has a GIL, but their STM (Software Transactional Memory) branch explored alternatives.

---

#### Garbage Collector — Deep Dive

**Primary mechanism**: Reference counting. Every object has a `ob_refcnt`. When it hits 0, memory is freed instantly.

**Secondary mechanism**: Generational cyclic GC. Three generations (0, 1, 2). New objects start in gen 0. If they survive a GC cycle, they promote to the next gen. Older gens are collected less frequently.

Cyclic GC only handles **reference cycles** (A→B→A). Simple reference counting handles everything else.

```python
import gc, sys

a = []
b = []
a.append(b)  # a -> b
b.append(a)  # b -> a  (circular reference!)

print(sys.getrefcount(a))  # shows reference count

del a, b     # refcount won't hit 0 — cyclic GC catches it
gc.collect() # force cyclic garbage collection

# GC tuning
gc.get_threshold()   # (700, 10, 10) — default thresholds
# gen0 collected every 700 allocations
# gen1 collected every 10 gen0 collections
# gen2 collected every 10 gen1 collections

gc.disable()  # turn off cyclic collector (for perf-critical code)
```

**`weakref` module** — creates references that don't increase refcount (avoids cycles in caches):

```python
import weakref

class ExpensiveObject:
    pass

obj = ExpensiveObject()
weak = weakref.ref(obj)

print(weak())    # <ExpensiveObject object>
del obj
print(weak())    # None — object was garbage collected
```

### 📌 TLDR for Interviews

> "The GIL is a mutex inside CPython that serializes bytecode execution across threads. It exists to protect reference counting (which isn't thread-safe). It switches threads every 5ms (`sys.setswitchinterval`). It's released during I/O and C extension calls, which is why threading works for I/O-bound tasks. For CPU parallelism, use `multiprocessing` (separate GIL per process). Python 3.13+ has an experimental free-threaded build (PEP 703) that removes the GIL entirely. Garbage collection uses reference counting + a 3-generation cyclic collector for circular references."

---

## 6. Iterators & Generators

> **📣 Interview-ready definition:** _"An iterator is any object that produces values one at a time via the `__iter__` and `__next__` methods. A generator is a special kind of iterator created with the `yield` keyword — it pauses its state between yields and resumes on each call. Generators are memory-efficient because they produce values lazily, on demand, instead of computing the whole sequence up front. A list of one million squares takes megabytes; a generator expression takes a few hundred bytes."_

### 🧑‍🍳 Layman Explanation

**Iterator** = A bookmark in a book. It remembers where you are and knows how to go to the next page. You can only go forward, one page at a time.

**Generator** = A lazy chef who only cooks the next dish when you ask for it, instead of cooking all 100 dishes upfront. Saves a ton of counter space (memory).

### 🔧 Technical Explanation

**Iterator Protocol:**

- Any object implementing `__iter__()` and `__next__()`.
- `__iter__()` returns the iterator object itself.
- `__next__()` returns the next item or raises `StopIteration`.

**Generator:**

- A function with `yield` instead of `return`. Calling it returns a generator object (an iterator).
- State is suspended at each `yield` and resumed on `next()`.
- **Lazy evaluation** — computes values on-demand, not upfront. Crucial for large datasets.
- `yield from` — delegates to a sub-generator (clean nesting).
- **Generator expression**: `(x*x for x in range(10**9))` — zero memory overhead.

```python
# Iterator class (manual)
class Countdown:
    def __init__(self, start):
        self.current = start
    def __iter__(self):
        return self
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# Generator function (much simpler)
def countdown(start):
    while start > 0:
        yield start
        start -= 1

# Memory comparison
import sys
list_comp = [x*x for x in range(1_000_000)]     # ~8 MB in memory
gen_expr  = (x*x for x in range(1_000_000))      # ~200 bytes!
print(sys.getsizeof(list_comp))  # 8448728
print(sys.getsizeof(gen_expr))   # 200
```

**`send()` and generator coroutines:**

```python
def accumulator():
    total = 0
    while True:
        value = yield total  # receives value via .send()
        total += value

acc = accumulator()
next(acc)        # prime the generator
acc.send(10)     # returns 10
acc.send(20)     # returns 30
```

### 📌 TLDR for Interviews

> "Generators use `yield` to lazily produce values one at a time — they're memory-efficient iterators. A list of 1M items takes megabytes; a generator expression takes ~200 bytes. Use generators for pipelines, streaming large files, and infinite sequences. They implement the iterator protocol (`__iter__` + `__next__`) automatically."

---

## 7. Concurrency vs Parallelism

> **📣 Interview-ready definition:** *"Concurrency is about *structure* — managing many tasks that can overlap in time, like one person juggling three balls. Parallelism is about *execution* — actually doing many tasks at the exact same moment, like three people each holding a ball. Concurrency works on a single core (asyncio); parallelism requires multiple cores (multiprocessing). They're related but distinct, and a system can be concurrent without being parallel — or both at once."*

### 🧑‍🍳 Layman Explanation

**Concurrency** = One person juggling 3 balls. Only one ball is in their hand at any moment, but they _manage_ all three and switch between them so fast it looks simultaneous.

**Parallelism** = Three people, each holding one ball. They all act at the exact same time. True simultaneous execution.

Concurrency is about **structure** (dealing with many things). Parallelism is about **execution** (doing many things).

### 🔧 Technical Explanation

|             | Concurrency                              | Parallelism                              |
| ----------- | ---------------------------------------- | ---------------------------------------- |
| Definition  | Managing multiple tasks that can overlap | Executing multiple tasks simultaneously  |
| Requires    | Single core is enough                    | Multiple cores/processors                |
| Python tool | `asyncio`, `threading`                   | `multiprocessing`, `ProcessPoolExecutor` |
| Analogy     | Interleaved execution                    | Simultaneous execution                   |
| Goal        | Responsiveness, throughput for I/O       | Raw speed for computation                |

```
Concurrency (single core, interleaved):
Task A: ██░░██░░██
Task B: ░░██░░██░░

Parallelism (multi-core, simultaneous):
Core 1 - Task A: ██████████
Core 2 - Task B: ██████████
```

- **Concurrent but not parallel**: `asyncio` event loop on a single thread.
- **Parallel but not concurrent**: Matrix multiplication spread across GPU cores (each core does one thing).
- **Both**: A web server with multiple worker processes, each running an async event loop.

### 📌 TLDR for Interviews

> "Concurrency is about _structuring_ a program to handle multiple tasks (switching between them). Parallelism is about _executing_ tasks at the same time on multiple cores. Python's `asyncio` is concurrent (single-threaded). `multiprocessing` is parallel. A production FastAPI app often uses both — multiple Uvicorn workers (parallel) each running an async event loop (concurrent)."

---

## 8. Async in Python & FastAPI

> **📣 Interview-ready definition:** _"`async def` defines a coroutine — a function that can pause and resume. `await` is the pause point, where the function yields control back to an event loop until the awaited operation (typically I/O) completes. The event loop is a single-threaded scheduler that juggles thousands of coroutines, switching between them on every `await`. FastAPI is built on this async foundation — it uses ASGI instead of WSGI, allowing it to handle massive concurrent I/O without spawning threads."_

### 🧑‍🍳 Layman Explanation

Imagine ordering at a restaurant. **Synchronous** = you stand at the counter, wait for your food, then sit down. Nobody else can order until you're done. **Async** = you place your order, get a buzzer, go sit down, and the buzzer rings when food is ready. Meanwhile, the counter takes other orders.

Think of `async def` as **"this task might need to wait for something."** And `await` as **"I'm waiting now — go serve someone else and come back to me when my thing is ready."**

FastAPI is a restaurant designed from the ground up for buzzers (async). It handles thousands of orders with a tiny staff.

### 🔧 Technical Explanation

---

#### What is a Coroutine?

A **coroutine** is a function that can be paused mid-execution and resumed later. It's the building block of async Python.

```python
# This is a coroutine FUNCTION (note: async def)
async def fetch_user(user_id):
    print("Starting fetch...")
    user = await db.get_user(user_id)   # PAUSE here, resume when DB responds
    print("Got user!")
    return user

# Calling it does NOT execute it — it returns a coroutine OBJECT
coro = fetch_user(42)
print(type(coro))  # <class 'coroutine'>

# To actually RUN it, you need an event loop:
result = asyncio.run(coro)
```

**Interview-friendly explanation:** "Calling `async def fetch_user()` doesn't run the function — it creates a coroutine object, like a recipe card. You need the event loop to actually cook it. `asyncio.run()` or `await` are what kick off execution."

---

#### What does `await` actually DO?

`await` does TWO things:

1. **Yields control** back to the event loop ("I'm going to wait, go do something else").
2. **Resumes** the coroutine when the awaited thing completes.

```python
# Think of await as a "bookmark + yield" in one keyword

async def make_breakfast():
    # Step 1: Start toasting bread (I/O-bound — waiting on toaster)
    toast = await toast_bread()       # PAUSE — event loop runs other tasks
                                       # RESUME — toast is ready

    # Step 2: Start brewing coffee (I/O-bound — waiting on machine)
    coffee = await brew_coffee()      # PAUSE — event loop runs other tasks
                                       # RESUME — coffee is ready

    return {"toast": toast, "coffee": coffee}
    # Problem: this is SEQUENTIAL — coffee waits for toast to finish!
```

**The KEY insight** — `await` is sequential within a coroutine. For TRUE concurrency, you need `gather()` or `create_task()`:

```python
async def make_breakfast_fast():
    # Fire both AT THE SAME TIME, wait for both to finish
    toast, coffee = await asyncio.gather(
        toast_bread(),      # starts immediately
        brew_coffee(),      # starts immediately, doesn't wait for toast
    )
    return {"toast": toast, "coffee": coffee}
    # Total time ≈ max(toast_time, coffee_time), NOT sum!
```

**Interview-ready diagram:**

```
SEQUENTIAL (two awaits):
toast_bread:  |████████████|
brew_coffee:                |████████████|
Total:        |████████████████████████████|  = 6s + 4s = 10s

CONCURRENT (gather):
toast_bread:  |████████████|
brew_coffee:  |████████████|
Total:        |████████████|                  = max(6s, 4s) = 6s
```

---

#### What can you `await`?

You can only `await` an **awaitable** object. Three types:

1. **Coroutines** — from `async def` functions
2. **Tasks** — from `asyncio.create_task()`
3. **Futures** — low-level, rarely used directly

```python
# ✅ You CAN await these
await asyncio.sleep(1)                          # coroutine
await httpx.AsyncClient().get("https://...")     # coroutine
task = asyncio.create_task(some_coroutine())
await task                                       # Task

# ❌ You CANNOT await these
await time.sleep(1)        # WRONG — time.sleep is sync, BLOCKS the event loop!
await requests.get(url)    # WRONG — requests is sync, use httpx/aiohttp instead
await some_regular_func()  # WRONG — regular functions aren't awaitable
```

**Critical mistake to call out in interviews:**

```python
# 🚨 THE #1 ASYNC BUG: Using sync I/O inside async functions
@app.get("/users")
async def get_users():
    # This BLOCKS the entire event loop! No other request can be served!
    users = requests.get("https://api.example.com/users")  # ❌ SYNC!

    # Correct: use an async HTTP client
    async with httpx.AsyncClient() as client:
        users = await client.get("https://api.example.com/users")  # ✅ ASYNC!
```

---

#### The Event Loop — How it works under the hood

The event loop is a **single-threaded scheduler** that manages all coroutines. Think of it as a while-loop that keeps checking: "Is anything ready? Is anything ready?"

```python
# Simplified mental model of what the event loop does:
while tasks_exist:
    for task in ready_tasks:
        run_task_until_next_await(task)      # execute until it hits "await"

    completed_io = check_os_for_completed_io()  # epoll/kqueue/select
    for io_event in completed_io:
        mark_waiting_task_as_ready(io_event)   # wake up the coroutine

    # Repeat forever until all tasks are done
```

**Real architecture:**

```
┌──────────────────────────────────────────────────┐
│                  EVENT LOOP                        │
│                                                    │
│   Ready Queue: [coroutine_A, coroutine_C]         │
│                                                    │
│   Waiting on I/O:                                  │
│     - coroutine_B → waiting on socket.recv()       │
│     - coroutine_D → waiting on DB response         │
│                                                    │
│   OS I/O Selector (epoll/kqueue):                  │
│     monitors file descriptors for readiness        │
│                                                    │
│   Timer heap:                                      │
│     - coroutine_E → wake up in 5 seconds           │
└──────────────────────────────────────────────────┘

Loop cycle:
1. Run coroutine_A until it awaits → moves to waiting
2. Run coroutine_C until it awaits → moves to waiting
3. Check OS selector → socket for B is ready → B moves to ready queue
4. Check timers → nothing due yet
5. Run coroutine_B until it awaits again or completes
6. Repeat...
```

---

#### `create_task()` vs `gather()` vs `wait()` — When to use what

```python
import asyncio

# 1. create_task() — fire and forget, or await later
async def background_example():
    task = asyncio.create_task(send_email())  # starts IMMEDIATELY in background
    # ... do other work while email sends ...
    result = await task  # optionally await it later

# 2. gather() — run multiple coroutines, wait for ALL
async def fetch_all():
    results = await asyncio.gather(
        fetch_api_1(),
        fetch_api_2(),
        fetch_api_3(),
        return_exceptions=True,  # don't crash if one fails — return exception obj
    )
    # results = [result1, result2, result3] — same order as input

# 3. wait() — more control: get results as they complete
async def fetch_with_control():
    tasks = {asyncio.create_task(fetch(url)) for url in urls}
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED,  # or ALL_COMPLETED, FIRST_EXCEPTION
    )
    for task in done:
        print(task.result())

# 4. as_completed() — process results in completion order
async def stream_results():
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    for coro in asyncio.as_completed(tasks):
        result = await coro  # yields in order of completion, not submission
        print(f"Got result: {result}")

# 5. TaskGroup (Python 3.11+) — structured concurrency
async def structured_example():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch_api_1())
        task2 = tg.create_task(fetch_api_2())
    # Both done here. If any raised, ALL are cancelled + ExceptionGroup raised.
    print(task1.result(), task2.result())
```

**Interview-friendly comparison:**

| Tool                | Use when                              | Cancellation                       | Error handling           |
| ------------------- | ------------------------------------- | ---------------------------------- | ------------------------ |
| `create_task()`     | Fire-and-forget or deferred await     | Manual                             | Manual                   |
| `gather()`          | Wait for ALL results, order matters   | Cancel all if one fails (optional) | `return_exceptions=True` |
| `wait()`            | Need first-completed or timeout logic | Manual on pending                  | Check `task.exception()` |
| `as_completed()`    | Process results as they arrive        | Manual                             | Per-task try/except      |
| `TaskGroup` (3.11+) | Structured concurrency, auto-cleanup  | Auto-cancels siblings on error     | `ExceptionGroup`         |

---

#### Async Context Managers and Async Iterators

```python
# Async Context Manager — cleanup for async resources
class AsyncDBPool:
    async def __aenter__(self):
        self.pool = await create_pool()
        return self.pool

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.pool.close()

async def main():
    async with AsyncDBPool() as pool:
        await pool.execute("SELECT ...")

# Async Iterator — async for loop
class AsyncPaginator:
    def __init__(self, url):
        self.url = url
        self.page = 0

    def __aiter__(self):
        return self

    async def __anext__(self):
        self.page += 1
        data = await httpx.AsyncClient().get(f"{self.url}?page={self.page}")
        items = data.json().get("items", [])
        if not items:
            raise StopAsyncIteration
        return items

# Usage
async for page_items in AsyncPaginator("https://api.example.com/users"):
    process(page_items)

# Async Generator (simpler version of above)
async def paginate(url):
    page = 0
    while True:
        page += 1
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{url}?page={page}")
        items = resp.json().get("items", [])
        if not items:
            break
        yield items  # async generator!

async for items in paginate("https://api.example.com/users"):
    process(items)
```

---

#### Common Async Pitfalls (Interview Gold)

```python
# Pitfall 1: Forgetting to await
async def bad():
    result = fetch_user(42)    # just creates coroutine, never runs!
    print(result)              # <coroutine object fetch_user at 0x...>
    # Fix: result = await fetch_user(42)

# Pitfall 2: Blocking the event loop
async def very_bad():
    time.sleep(10)             # BLOCKS everything for 10 seconds!
    # Fix: await asyncio.sleep(10)

    data = requests.get(url)   # BLOCKS! Use httpx/aiohttp instead
    # Fix: async with httpx.AsyncClient() as c: data = await c.get(url)

    result = heavy_cpu_work()  # BLOCKS! CPU-bound in async = disaster
    # Fix: loop = asyncio.get_event_loop()
    #       result = await loop.run_in_executor(None, heavy_cpu_work)

# Pitfall 3: Creating tasks that nobody awaits (fire-and-forget dangers)
async def leaky():
    asyncio.create_task(send_email())  # if main() exits, this may be cancelled!
    # Fix: keep a reference and await it, or use TaskGroup

# Pitfall 4: Sequential awaits when you could be concurrent
async def slow():
    a = await fetch_api_1()    # waits 2s
    b = await fetch_api_2()    # waits 2s
    # Total: 4s

async def fast():
    a, b = await asyncio.gather(fetch_api_1(), fetch_api_2())
    # Total: 2s
```

---

#### FastAPI — Async in Practice

```python
from fastapi import FastAPI, Depends, BackgroundTasks
import httpx
import asyncio

app = FastAPI()

# --- Async dependency injection ---
async def get_db():
    db = await connect_to_db()
    try:
        yield db  # async generator as a dependency!
    finally:
        await db.close()

@app.get("/users/{user_id}")
async def get_user(user_id: int, db=Depends(get_db)):
    return await db.fetch_one("SELECT * FROM users WHERE id = $1", user_id)

# --- Parallel external API calls ---
@app.get("/dashboard")
async def dashboard():
    async with httpx.AsyncClient() as client:
        # Fire 3 requests concurrently — total time = slowest one
        orders, payments, notifications = await asyncio.gather(
            client.get("http://order-service/api/orders"),
            client.get("http://payment-service/api/payments"),
            client.get("http://notification-service/api/recent"),
        )
    return {
        "orders": orders.json(),
        "payments": payments.json(),
        "notifications": notifications.json(),
    }

# --- Background tasks (fire-and-forget done RIGHT) ---
@app.post("/register")
async def register_user(user: UserCreate, bg: BackgroundTasks):
    new_user = await create_user(user)
    bg.add_task(send_welcome_email, new_user.email)  # runs AFTER response sent
    bg.add_task(notify_slack, f"New user: {new_user.name}")
    return new_user  # response returned immediately, emails send in background

# --- When to use sync def vs async def in FastAPI ---
# Use async def when:
#   - You're calling async libraries (httpx, asyncpg, aioredis)
#   - You're doing I/O that has async versions available

# Use plain def when:
#   - You're calling sync libraries (requests, psycopg2, pandas)
#   - FastAPI auto-runs it in a threadpool — doesn't block the event loop
#   - This is SAFE and often SIMPLER than forcing everything to be async

@app.get("/report")
def generate_report():
    # sync is FINE here — FastAPI handles it in a threadpool
    df = pandas.read_sql("SELECT * FROM sales", sync_db_connection)
    return df.to_dict()
```

**FastAPI async architecture in production:**

```
                    ┌─────────────────────────────────────┐
                    │           Nginx / Traefik            │
                    │         (reverse proxy, TLS)         │
                    └─────────────┬───────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
     ┌────────▼──────┐  ┌────────▼──────┐  ┌────────▼──────┐
     │  Uvicorn W1   │  │  Uvicorn W2   │  │  Uvicorn W3   │
     │  (Process 1)  │  │  (Process 2)  │  │  (Process 3)  │
     │               │  │               │  │               │
     │  Event Loop   │  │  Event Loop   │  │  Event Loop   │
     │  ~1000 async  │  │  ~1000 async  │  │  ~1000 async  │
     │  connections  │  │  connections  │  │  connections  │
     └───────────────┘  └───────────────┘  └───────────────┘
              │                   │                   │
              ▼                   ▼                   ▼
     ┌─────────────────────────────────────────────────────┐
     │  asyncpg pool / Redis / httpx (async I/O clients)   │
     └─────────────────────────────────────────────────────┘

     3 workers × ~1000 concurrent connections = ~3000 requests handled
     Each worker: 1 process, 1 thread, 1 event loop, MANY coroutines
```

> "`async def` creates a coroutine — a pausable function. `await` pauses the coroutine and gives control back to the event loop until the I/O completes. The event loop is a single-threaded scheduler that multiplexes thousands of coroutines. Use `gather()` for parallel I/O calls, `create_task()` for background work, and `TaskGroup` (3.11+) for structured concurrency. In FastAPI, `async def` runs on the event loop, plain `def` auto-runs in a threadpool. Never use sync I/O (requests, time.sleep) inside `async def` — it blocks the entire event loop. In production, Uvicorn with multiple workers gives you parallelism (processes) × concurrency (async) for massive throughput."

---

## 9. Python Concepts Deep Dive

> **📣 Interview-ready definition:** *"This covers Python language fundamentals beyond the surface — how variables actually work (name bindings to objects, not containers), the difference between mutable and immutable types and the bugs that follow, the three kinds of copy (reference, shallow via `copy()`, deep via `deepcopy()`), slicing syntax, the four OOP pillars (Encapsulation, Inheritance, Polymorphism, Abstraction), magic dunder methods that integrate with Python's syntax, `*args`/`\*_kwargs`, comprehensions, type hints, and exception handling. These are the everyday tools — interviewers test them because they reveal how deeply you actually understand Python."_

This section covers all the Python language fundamentals interviewers love to drill — copy semantics, slicing, OOP pillars, magic methods, and idiomatic Python.

---

### 9.1 Variables, References, `is` vs `==`

> **📣 Definition:** _"In Python, variables are name tags pointing to objects in memory, not boxes that hold values. `==` checks if two values are equal; `is` checks if two names point to the exact same object in memory. Use `is` only for `None`, `True`, `False`. Python caches small integers (-5 to 256) and short strings, which can make `is` accidentally return True for equal values — never rely on it."_

**Layman:** A variable in Python is a **name tag stuck on an object**, not a box holding a value. `a = [1, 2]` doesn't create a variable that "contains" the list — it creates a list object somewhere in memory and labels it `a`.

**Technical:**

```python
a = [1, 2, 3]
b = a               # b labels the SAME list (not a copy!)
b.append(4)
print(a)            # [1, 2, 3, 4]   ← a sees the change!

# `is` checks identity (same object in memory?)
# `==` checks equality (same VALUES?)
x = [1, 2]
y = [1, 2]
print(x == y)       # True  — values are equal
print(x is y)       # False — different objects in memory

# Caching gotcha
a = 256
b = 256
print(a is b)       # True — Python caches small ints (-5 to 256)

a = 257
b = 257
print(a is b)       # False (or True depending on context — don't rely on this!)

# Always use `is` for: None, True, False
if x is None: ...   # ✅ correct
if x == None: ...   # ❌ works but unidiomatic
```

📌 **TLDR:** "Python variables are name bindings to objects, not containers. `is` checks identity (memory address), `==` checks equality. Use `is` only for `None`, `True`, `False`. Small integers and short strings may be cached — never rely on it for logic."

---

### 9.2 Mutable vs Immutable Types

> **📣 Definition:** _"Mutable types (list, dict, set) can be changed in place after creation; immutable types (int, float, str, tuple, frozenset) cannot — operations on them return new objects. This matters because immutable objects are safe to share and use as dict keys, while mutable objects can cause sneaky bugs when passed around or used as default arguments — the classic 'mutable default argument' trap creates a single shared list across all calls."_

| Mutable (can change)                | Immutable (cannot change)                                    |
| ----------------------------------- | ------------------------------------------------------------ |
| `list`, `dict`, `set`, `bytearray`  | `int`, `float`, `str`, `tuple`, `frozenset`, `bytes`, `bool` |
| Custom class instances (by default) | `namedtuple`, `dataclass(frozen=True)`                       |

```python
# Immutable — operations return NEW objects
s = "hello"
s2 = s.upper()      # s is unchanged, s2 is new
print(id(s), id(s2))   # different memory addresses

# Mutable — operations modify IN PLACE
lst = [1, 2, 3]
lst.append(4)       # same list, modified
print(id(lst))      # same id

# THE CLASSIC TRAP: mutable default argument
def add_item(item, items=[]):     # ❌ default list created ONCE at def time
    items.append(item)
    return items

print(add_item("a"))    # ['a']
print(add_item("b"))    # ['a', 'b']  ← surprise! same list reused!

# Fix
def add_item(item, items=None):   # ✅ create fresh list each call
    if items is None:
        items = []
    items.append(item)
    return items
```

📌 **TLDR:** "Lists, dicts, sets are mutable; ints, strs, tuples, frozensets are immutable. The classic trap: mutable default arguments are evaluated once at function definition — use `None` and create the mutable inside the function."

---

### 9.3 Copy Types — Shallow vs Deep

> **📣 Definition:** _"There are three ways to 'copy' an object in Python. **Reference assignment** (`b = a`) doesn't copy at all — both names point to the same object, so changes via either are visible to both. **Shallow copy** (`copy.copy()`, `list(a)`, `a[:]`) creates a new top-level container but the nested objects are still shared. **Deep copy** (`copy.deepcopy()`) recursively duplicates everything — fully independent, but slower."_

**Layman:** A **shallow copy** is like photocopying the cover of a folder — the cover is new, but it still points to the same papers inside. A **deep copy** photocopies the entire folder including all the papers inside, recursively.

**Technical:**

```python
import copy

original = [[1, 2, 3], [4, 5, 6]]

# 1. Reference (NOT a copy!)
ref = original
ref[0][0] = 99
print(original)     # [[99, 2, 3], [4, 5, 6]]   ← original changed!

# 2. Shallow copy — top-level new, but inner objects shared
shallow = copy.copy(original)        # or original.copy() or list(original) or original[:]
shallow[0][0] = 77
print(original)     # [[77, 2, 3], [4, 5, 6]]   ← inner list shared!

shallow.append([7, 8, 9])
print(original)     # NOT changed at top level — shallow is its own list

# 3. Deep copy — entire object graph cloned, no sharing
deep = copy.deepcopy(original)
deep[0][0] = 11
print(original)     # unchanged
```

**Visual:**

```
ORIGINAL:  outer_list ─→ [inner_list_1, inner_list_2]
                              │              │
                              ▼              ▼
                         [1, 2, 3]      [4, 5, 6]

SHALLOW COPY:
new_outer_list ─→ [inner_list_1, inner_list_2]  ← SAME inner lists!
                       │              │
                       ▼              ▼
                  [1, 2, 3]      [4, 5, 6]

DEEP COPY:
new_outer_list ─→ [new_inner_1, new_inner_2]   ← all new!
                       │              │
                       ▼              ▼
                  [1, 2, 3]      [4, 5, 6]
```

**Multiple ways to make a shallow copy of a list:**

```python
a = [1, 2, 3]
b = a.copy()                  # method
c = list(a)                   # constructor
d = a[:]                      # slicing
e = copy.copy(a)              # copy module
import operator
# All four produce shallow copies. Equivalent for flat lists.

# For dicts
d = {'a': 1, 'b': [1, 2]}
shallow = d.copy()            # or dict(d) or {**d}
```

**When deep copy is expensive:** `deepcopy` walks the entire object tree and is significantly slower. For large structures, consider serialization (pickle, json) or domain-specific copying.

📌 **TLDR:** "Reference assignment (`b = a`) doesn't copy. Shallow copy (`a.copy()`, `a[:]`, `list(a)`, `copy.copy()`) duplicates only the top level — nested objects are still shared. Deep copy (`copy.deepcopy()`) recursively duplicates everything but is slower. The classic interview question: 'What happens with `b = a; a.append(1)`?' — `b` sees the change because both labels point to the same list."

---

### 9.4 Slicing & Dicing

> **📣 Definition:** _"Slicing is Python's syntax for extracting parts of a sequence — `sequence[start:stop:step]`. The stop index is exclusive, negative indices count from the end, and `[::-1]` reverses any sequence. `[:]` makes a shallow copy. Slice assignment lets you replace parts of a list in place. Important nuance: Python's built-in slicing always returns a copy, but NumPy slicing returns a view (changes affect the original)."_

**Layman:** Slicing is taking a piece out of a sequence (list, string, tuple). Like cutting a slice from a loaf of bread — pick the start, end, and step.

**Technical — `[start:stop:step]`:**

```python
nums = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# Basic slicing
nums[2:5]       # [2, 3, 4]      — indices 2, 3, 4 (stop is EXCLUSIVE)
nums[:3]        # [0, 1, 2]      — start defaults to 0
nums[7:]        # [7, 8, 9]      — stop defaults to len
nums[:]         # [0..9]         — full shallow copy!
nums[-3:]       # [7, 8, 9]      — last 3 elements
nums[:-2]       # [0..7]         — all except last 2

# Step
nums[::2]       # [0, 2, 4, 6, 8]    — every 2nd element
nums[1::2]      # [1, 3, 5, 7, 9]    — odd indices
nums[::-1]      # [9, 8, ..., 0]     — REVERSE the list (interview classic!)
nums[::-2]      # [9, 7, 5, 3, 1]    — reverse, every 2nd

# String slicing (same syntax)
s = "Hello, World!"
s[7:12]         # "World"
s[::-1]         # "!dlroW ,olleH"   — palindrome check!

# Tuple slicing — same syntax
t = (1, 2, 3, 4, 5)
t[1:4]          # (2, 3, 4)

# Slice ASSIGNMENT (lists only — they're mutable)
nums[2:5] = ['a', 'b']    # replace 3 elements with 2 → nums = [0,1,'a','b',5,6,7,8,9]
nums[:] = []              # clear in place (vs nums = [] which rebinds)

# Common idioms
first_3 = nums[:3]
last_3 = nums[-3:]
without_first_last = nums[1:-1]
reversed_str = s[::-1]
every_2nd = nums[::2]

# Slice object — for storage/reuse
my_slice = slice(1, 5, 2)
nums[my_slice]            # equivalent to nums[1:5:2]
```

**Subtle gotcha — slicing creates copies!**

```python
big_list = list(range(10**6))
chunk = big_list[100:200]     # creates a NEW list of 100 elements
# For large lists, consider islice() to avoid copying:
from itertools import islice
chunk = list(islice(big_list, 100, 200))    # iterator-based, lazy
```

**NumPy slicing is different — returns views, not copies:**

```python
import numpy as np
arr = np.array([1, 2, 3, 4, 5])
view = arr[1:4]           # VIEW into arr's memory
view[0] = 99
print(arr)                # [1, 99, 3, 4, 5]  ← original changed!
```

📌 **TLDR:** "Slice syntax: `[start:stop:step]`. Stop is exclusive. Negative indices count from the end. `[::-1]` reverses any sequence. `[:]` makes a shallow copy. Slice assignment (`a[2:5] = [x,y]`) modifies lists in place. NumPy slicing returns a view (changes affect original) — Python's built-in slicing always returns a copy."

---

### 9.5 OOP Pillars — Encapsulation, Inheritance, Polymorphism, Abstraction

> **📣 Definition:** _"The four pillars of object-oriented programming. **Encapsulation** = bundling data + behavior into a class and controlling external access (Python uses naming conventions, not enforcement). **Inheritance** = a subclass reuses and extends a parent's behavior (IS-A relationships). **Polymorphism** = same interface, different implementations — achieved in Python via method overriding, duck typing, and operator overloading. **Abstraction** = hiding complexity behind a simple interface, often via abstract base classes (`abc.ABC`) or Protocols. The easy distinction: encapsulation hides data, abstraction hides complexity."_

The **four pillars** of OOP — every interviewer asks about these.

#### 1️⃣ Encapsulation

**Layman:** A medicine capsule hides the powder inside. You only interact with the outside (swallow it). The internal mechanism is hidden — and changing the formula doesn't affect how you take it.

**Technical:** Bundling data + methods that operate on that data into a class, while controlling external access.

```python
class BankAccount:
    def __init__(self, owner, initial_balance=0):
        self.owner = owner                  # public — anyone can read/write
        self._account_number = "AC123"      # protected (convention) — internal use
        self.__balance = initial_balance    # private (name-mangled) — hidden

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("Amount must be positive")
        self.__balance += amount            # internal logic, validated

    def withdraw(self, amount):
        if amount > self.__balance:
            raise ValueError("Insufficient funds")
        self.__balance -= amount

    def get_balance(self):                  # controlled READ access
        return self.__balance

# Usage
acc = BankAccount("Tushar", 1000)
acc.deposit(500)
print(acc.get_balance())     # 1500

# Direct access blocked (mostly)
print(acc.owner)             # ✅ "Tushar"
print(acc._account_number)   # ⚠️ "AC123" — accessible but convention says don't
print(acc.__balance)         # ❌ AttributeError!
print(acc._BankAccount__balance)   # ⚠️ accessible via name mangling — still possible
```

**Pythonic encapsulation with `@property`:**

```python
class Temperature:
    def __init__(self, celsius):
        self._celsius = celsius

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("Below absolute zero!")
        self._celsius = value

    @property
    def fahrenheit(self):                       # computed property
        return self._celsius * 9/5 + 32

# Usage looks like attribute access, but goes through methods
t = Temperature(25)
t.celsius = 30                                  # calls setter (with validation)
print(t.fahrenheit)                             # 86.0 — computed on the fly
t.celsius = -300                                # ValueError!
```

📌 **TLDR:** "Encapsulation = bundling data with the methods that act on it, hiding internal state behind a controlled interface. Python uses naming conventions: `_protected` (don't touch by convention), `__private` (name-mangled to `_ClassName__attr`). Use `@property` for Pythonic getters/setters with validation."

---

#### 2️⃣ Inheritance

**Layman:** A SUV inherits from Car (it IS a car with extras). You don't redefine "has wheels" — you inherit that. Only add what's new (4WD, larger frame).

**Technical:**

```python
# Single inheritance
class Animal:
    def __init__(self, name):
        self.name = name

    def speak(self):
        return "Some sound"

class Dog(Animal):                 # inherits from Animal
    def speak(self):                # override
        return "Woof!"

    def fetch(self):                # extend
        return f"{self.name} fetches the ball"

# Multiple inheritance + MRO (Method Resolution Order)
class A:
    def hello(self): return "A"

class B(A):
    def hello(self): return "B → " + super().hello()

class C(A):
    def hello(self): return "C → " + super().hello()

class D(B, C):
    def hello(self): return "D → " + super().hello()

print(D().hello())          # "D → B → C → A"
print(D.__mro__)            # (D, B, C, A, object) — C3 linearization

# Mixin pattern (composition via inheritance)
class TimestampMixin:
    def get_age(self):
        from datetime import datetime
        return datetime.now() - self.created_at

class SerializableMixin:
    def to_dict(self):
        return {k: v for k, v in self.__dict__.items()}

class User(TimestampMixin, SerializableMixin):
    def __init__(self, name):
        from datetime import datetime
        self.name = name
        self.created_at = datetime.now()

u = User("Tushar")
print(u.to_dict())          # from SerializableMixin
print(u.get_age())          # from TimestampMixin
```

📌 **TLDR:** "Inheritance enables IS-A relationships and code reuse. Python supports multiple inheritance via C3 MRO. Use mixins for composing behavior. Prefer composition over deep inheritance hierarchies — `has-a` over `is-a` is often cleaner."

---

#### 3️⃣ Polymorphism

**Layman:** Same method name, different behaviors. Like a USB port — plug in a phone, mouse, or keyboard. The port (`use()`) works the same way; what happens depends on the device.

**Technical:**

```python
# 1. Method overriding (runtime polymorphism)
class Shape:
    def area(self):
        raise NotImplementedError

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14 * self.r ** 2

class Square(Shape):
    def __init__(self, s): self.s = s
    def area(self): return self.s ** 2

shapes = [Circle(5), Square(3), Circle(2)]
for shape in shapes:
    print(shape.area())     # SAME call, DIFFERENT behavior per shape

# 2. Duck typing — Python's preferred polymorphism
# "If it walks like a duck and quacks like a duck, it's a duck"
class Duck:
    def quack(self): return "Quack!"

class Dog:
    def quack(self): return "Woof! (pretending to be a duck)"

def make_it_quack(thing):
    return thing.quack()    # doesn't care about type — only that .quack() exists

print(make_it_quack(Duck()))    # works
print(make_it_quack(Dog()))     # also works! Python doesn't enforce inheritance.

# 3. Operator overloading — polymorphism for operators
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):           # overloads `+`
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):          # overloads `*`
        return Vector(self.x * scalar, self.y * scalar)

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)              # Vector(4, 6)
print(v1 * 3)               # Vector(3, 6)

# 4. Function/method overloading — Python doesn't have it natively!
# Use default args, *args, or functools.singledispatch:
from functools import singledispatch

@singledispatch
def process(arg):
    raise NotImplementedError("Unknown type")

@process.register(int)
def _(arg):
    return f"Processing int: {arg}"

@process.register(str)
def _(arg):
    return f"Processing string: {arg}"

@process.register(list)
def _(arg):
    return f"Processing list of {len(arg)} items"

print(process(42))           # Processing int: 42
print(process("hi"))         # Processing string: hi
print(process([1, 2]))       # Processing list of 2 items
```

📌 **TLDR:** "Polymorphism = same interface, different implementations. Achieved via method overriding (subclasses), duck typing (Python's preferred way — type doesn't matter, only behavior), operator overloading (`__add__`, `__mul__`, etc.), and `functools.singledispatch` for function overloading. Python doesn't have C++/Java-style method overloading by signature — use default args or `*args`."

---

#### 4️⃣ Abstraction

**Layman:** Driving a car, you press the gas pedal — you don't need to know about combustion, fuel injection, or pistons. The pedal is the **abstraction** — a simplified interface to a complex system.

**Technical:**

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):              # abstract class — can't instantiate!
    @abstractmethod
    def process(self, amount): pass

    @abstractmethod
    def refund(self, transaction_id): pass

    def log(self, msg):                   # concrete method available to all subclasses
        print(f"[Log] {msg}")

class StripeProcessor(PaymentProcessor):
    def process(self, amount):
        self.log(f"Stripe charging {amount}")
        # Stripe-specific implementation

    def refund(self, transaction_id):
        self.log(f"Stripe refunding {transaction_id}")

class RazorpayProcessor(PaymentProcessor):
    def process(self, amount):
        self.log(f"Razorpay charging {amount}")

    def refund(self, transaction_id):
        self.log(f"Razorpay refunding {transaction_id}")

# PaymentProcessor()              # ❌ TypeError: Can't instantiate abstract class
processor = StripeProcessor()     # ✅ works — all abstract methods implemented
processor.process(500)

# Caller doesn't care which payment system — they only see the abstraction
def checkout(processor: PaymentProcessor, amount: float):
    processor.process(amount)     # works for ANY PaymentProcessor subclass
```

**Protocols (PEP 544 — structural typing, Python 3.8+):**

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

class Circle:
    def draw(self): print("○")

class Square:
    def draw(self): print("□")

def render(shape: Drawable):      # accepts ANYTHING with draw() — duck typing + types
    shape.draw()

render(Circle())     # ✅
render(Square())     # ✅
# Neither inherits from Drawable — Python checks structurally.
```

**Differences between abstraction & encapsulation (interview gotcha):**

- **Encapsulation** = hiding **internal state** (HOW the object stores data).
- **Abstraction** = hiding **complexity** behind a simpler interface (WHAT the object does).
- Often confused — the easy way: encapsulation is about **data**, abstraction is about **behavior**.

📌 **TLDR:** "Abstraction = hiding complexity behind a simpler interface. Achieved with abstract base classes (`abc.ABC` + `@abstractmethod`) or Protocols (PEP 544 — structural typing). Forces subclasses to implement key methods. Encapsulation hides DATA; abstraction hides COMPLEXITY."

---

### 9.6 Magic / Dunder Methods (`__xxx__`)

> **📣 Definition:** _"Magic methods (also called dunder methods, short for 'double underscore') let your custom classes integrate with Python's built-in syntax and operators. `__init__` is the constructor, `__str__` and `__repr__` control how an object is displayed, `__eq__` and `__hash__` make objects comparable and usable in sets/dicts, `__add__`/`__mul__` overload operators, `__iter__`/`__next__` make objects iterable, and `__enter__`/`__exit__` make them context managers. They're the protocol behind 'Pythonic' integration."_

These let your custom classes integrate with Python's syntax (operators, iteration, context managers, etc.).

| Method                                         | Purpose                             | Example use                                       |
| ---------------------------------------------- | ----------------------------------- | ------------------------------------------------- |
| `__init__`                                     | Constructor                         | `obj = MyClass(...)`                              |
| `__new__`                                      | Object creation (before `__init__`) | Singletons, immutable types                       |
| `__del__`                                      | Destructor                          | Resource cleanup (rare — prefer context managers) |
| `__repr__`                                     | Unambiguous string repr (for devs)  | `repr(obj)`, REPL display                         |
| `__str__`                                      | Human-readable string               | `str(obj)`, `print(obj)`                          |
| `__eq__`, `__ne__`                             | `==`, `!=`                          | Custom equality                                   |
| `__lt__`, `__le__`, `__gt__`, `__ge__`         | `<`, `<=`, `>`, `>=`                | Sorting, ordering                                 |
| `__hash__`                                     | Hashing                             | Use as dict key or in set                         |
| `__len__`                                      | `len(obj)`                          | Custom collections                                |
| `__getitem__`, `__setitem__`, `__delitem__`    | `obj[k]`, `obj[k]=v`, `del obj[k]`  | Custom indexing                                   |
| `__iter__`, `__next__`                         | `for x in obj`                      | Iterators                                         |
| `__contains__`                                 | `x in obj`                          | Membership test                                   |
| `__call__`                                     | `obj()`                             | Callable instances                                |
| `__enter__`, `__exit__`                        | `with obj as x:`                    | Context managers                                  |
| `__add__`, `__sub__`, `__mul__`, `__truediv__` | `+`, `-`, `*`, `/`                  | Arithmetic                                        |
| `__getattr__`, `__setattr__`, `__delattr__`    | Attribute access                    | Dynamic attrs, proxies                            |
| `__bool__`                                     | `bool(obj)` / `if obj:`             | Truthiness                                        |

**Comprehensive example:**

```python
class Money:
    def __init__(self, amount, currency="INR"):
        self.amount = amount
        self.currency = currency

    def __repr__(self):                       # for developers (debug)
        return f"Money({self.amount}, '{self.currency}')"

    def __str__(self):                        # for users (print)
        return f"₹{self.amount:.2f}" if self.currency == "INR" else f"{self.amount} {self.currency}"

    def __eq__(self, other):
        return self.amount == other.amount and self.currency == other.currency

    def __lt__(self, other):
        if self.currency != other.currency:
            raise ValueError("Cannot compare different currencies")
        return self.amount < other.amount

    def __add__(self, other):
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)

    def __mul__(self, scalar):                # for scaling
        return Money(self.amount * scalar, self.currency)

    def __hash__(self):
        return hash((self.amount, self.currency))

    def __bool__(self):
        return self.amount != 0

# All this works!
a = Money(500)
b = Money(300)
print(a + b)              # ₹800.00
print(a > b)              # True
print(a * 2)              # ₹1000.00
print(bool(Money(0)))     # False
print({a, b})             # set works because __hash__ defined
```

📌 **TLDR:** "Magic methods (dunder methods) integrate custom classes with Python's syntax — operators (`+` via `__add__`), iteration (`__iter__`), context managers (`__enter__`/`__exit__`), comparisons (`__eq__`, `__lt__`), etc. Always pair `__eq__` with `__hash__` if you want the object to be usable in sets/dicts."

---

### 9.7 `*args`, `**kwargs`, and Argument Unpacking

> **📣 Definition:** *"`*args`collects extra positional arguments into a tuple;`\*_kwargs`collects extra keyword arguments into a dict. The same syntax at the call site does the opposite — unpacks an iterable or dict into individual arguments. Use`_`alone in a signature to enforce keyword-only arguments after it; use`/` (Python 3.8+) to enforce positional-only before it. This pattern is what makes decorators and wrapper functions possible — they forward unknown args to the wrapped function."\*

```python
# *args — variable positional arguments → tuple
def add_all(*args):
    return sum(args)
add_all(1, 2, 3)            # 6
add_all()                   # 0

# **kwargs — variable keyword arguments → dict
def show_kwargs(**kwargs):
    for k, v in kwargs.items():
        print(f"{k}={v}")
show_kwargs(name="Tushar", role="Founder")

# Combine both
def flexible(positional, *args, default="x", **kwargs):
    print(positional, args, default, kwargs)
flexible(1, 2, 3, default="y", extra=True)
# 1 (2, 3) y {'extra': True}

# Unpacking AT CALL site
nums = [1, 2, 3]
add_all(*nums)              # equivalent to add_all(1, 2, 3)

config = {"timeout": 5, "retries": 3}
def connect(timeout, retries):
    pass
connect(**config)           # equivalent to connect(timeout=5, retries=3)

# Forwarding args (common pattern in decorators/wrappers)
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)        # forward EVERYTHING
    return wrapper

# Keyword-only arguments (after `*`)
def configure(host, port, *, debug=False, ssl=False):
    # debug and ssl MUST be passed as keywords
    pass
configure("localhost", 8080, debug=True)        # ✅
configure("localhost", 8080, True, True)        # ❌ TypeError

# Positional-only arguments (Python 3.8+, after `/`)
def divide(a, b, /):
    return a / b
divide(10, 2)               # ✅
divide(a=10, b=2)           # ❌ TypeError — a, b are positional-only
```

📌 **TLDR:** "`*args` collects extra positional args into a tuple; `**kwargs` collects extra keyword args into a dict. Use `*` to enforce keyword-only args after it, `/` to enforce positional-only before it. `*` and `**` at the call site UNPACK iterables/dicts into args."

---

### 9.8 Lambda, Map, Filter, Reduce

> **📣 Definition:** _"`lambda` creates anonymous one-line functions, useful as `key=` arguments for sorting or short predicates. `map` applies a function to each item, `filter` keeps items matching a predicate, `reduce` folds a sequence into a single value. In modern idiomatic Python, list/dict/set comprehensions usually replace `map` and `filter` for readability, and built-ins like `sum`, `max`, `min` replace `reduce` for common cases. Reach for these tools sparingly — they're more functional-programming style than typical Python."_

```python
# Lambda — anonymous, single-expression function
square = lambda x: x ** 2
print(square(5))            # 25

# Equivalent to:
def square(x): return x ** 2

# When to use lambda: only for tiny one-liners passed to other functions
sorted_data = sorted([(1, 'b'), (2, 'a')], key=lambda t: t[1])

# Map — apply a function to every item
nums = [1, 2, 3, 4]
squared = list(map(lambda x: x**2, nums))           # [1, 4, 9, 16]
# Pythonic alternative — list comprehension:
squared = [x**2 for x in nums]                      # ✅ more readable

# Filter — keep items where predicate is True
evens = list(filter(lambda x: x % 2 == 0, nums))    # [2, 4]
# Pythonic:
evens = [x for x in nums if x % 2 == 0]             # ✅

# Reduce — fold a sequence into a single value
from functools import reduce
total = reduce(lambda acc, x: acc + x, nums, 0)     # 10
# Pythonic:
total = sum(nums)                                   # ✅ when builtin exists

# When reduce is genuinely useful — non-trivial accumulation
from functools import reduce
words = ["hello", "world", "python"]
longest = reduce(lambda a, b: a if len(a) > len(b) else b, words)
# 'python'
```

📌 **TLDR:** "Lambda creates anonymous one-line functions. `map`, `filter`, `reduce` are functional-programming staples — but in idiomatic Python, prefer list/dict/set comprehensions for `map`/`filter`, and built-ins (`sum`, `max`, `min`) over `reduce`. Reach for lambda only as a quick `key=` argument or a small predicate."

---

### 9.9 Comprehensions — List, Dict, Set, Generator

> **📣 Definition:** *"Comprehensions are Pythonic shortcuts for building lists, dicts, or sets in a single expression: `[x*2 for x in nums if x > 0]`. They're more readable than `map`/`filter` and faster than equivalent for-loops because they're optimized in CPython. Generator expressions use parentheses instead of brackets — same syntax, but lazy: they yield values on demand without building a full collection in memory, which matters hugely for large datasets."\*

```python
# List comprehension
squares = [x**2 for x in range(10)]
# [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# With filter
evens_squared = [x**2 for x in range(10) if x % 2 == 0]
# [0, 4, 16, 36, 64]

# Nested
matrix = [[1,2,3], [4,5,6], [7,8,9]]
flattened = [x for row in matrix for x in row]
# [1, 2, 3, 4, 5, 6, 7, 8, 9]

# Conditional expression IN the value
labels = ["even" if x % 2 == 0 else "odd" for x in range(5)]
# ['even', 'odd', 'even', 'odd', 'even']

# Dict comprehension
squares_map = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Invert a dict
d = {'a': 1, 'b': 2}
inverted = {v: k for k, v in d.items()}
# {1: 'a', 2: 'b'}

# Set comprehension
unique_lengths = {len(word) for word in ["hi", "hello", "hey", "hello"]}
# {2, 3, 5}

# Generator expression — lazy, memory-efficient
total = sum(x**2 for x in range(10**6))     # no list created — streamed
big_list = [x**2 for x in range(10**6)]     # creates a 1M-element list in memory
```

📌 **TLDR:** "Comprehensions are Pythonic shortcuts for building lists/dicts/sets. Generator expressions `(x for x in ...)` are lazy and memory-efficient — use them for one-shot iteration over large data. Avoid deeply nested comprehensions — switch to a regular for-loop when readability suffers."

---

### 9.10 Type Hints & Modern Python Typing

> **📣 Definition:** _"Type hints let you annotate function signatures and variables with expected types — `def add(x: int, y: int) -> int`. Python doesn't enforce them at runtime; they're checked by static analyzers like `mypy` or `pyright` before deployment. Modern syntax (3.10+) uses `int | None` instead of `Optional[int]` and `list[int]` instead of `List[int]`. Advanced features include `Protocol` for structural typing (duck typing with type safety), `TypedDict` for dict shapes, and `TypeVar` for generic functions. FastAPI and Pydantic use type hints for runtime validation, blending the two worlds."_

```python
from typing import Optional, Union, List, Dict, Tuple, Callable, Any, TypeVar, Generic

# Basic type hints
def greet(name: str, age: int = 18) -> str:
    return f"Hello {name}, age {age}"

# Optional = Union[X, None]
def find_user(user_id: int) -> Optional[dict]:
    return db.get(user_id)          # returns dict or None

# Python 3.10+ shorthand (preferred):
def find_user(user_id: int) -> dict | None: ...

# Lists, dicts (Python 3.9+ uses lowercase generics)
def process(items: list[int]) -> dict[str, int]: ...

# Callable
def retry(fn: Callable[[int, str], bool]) -> bool:
    return fn(1, "hello")

# Generics
T = TypeVar("T")

def first(items: list[T]) -> T | None:
    return items[0] if items else None

# Protocols — duck typing with type hints (PEP 544)
from typing import Protocol

class HasLength(Protocol):
    def __len__(self) -> int: ...

def is_empty(obj: HasLength) -> bool:
    return len(obj) == 0

# TypedDict — typed dict structure
from typing import TypedDict

class UserDict(TypedDict):
    id: int
    name: str
    email: str

def create_user(data: UserDict) -> None: ...

# Runtime nothing changes — static type checkers (mypy, pyright) verify
```

📌 **TLDR:** "Type hints are compile-time only — Python doesn't enforce them at runtime. Tools like `mypy`/`pyright` catch errors before deploy. Modern syntax: `list[int]` not `List[int]`, `int | None` not `Optional[int]` (3.10+). Use `Protocol` for structural typing, `TypedDict` for typed dict shapes, `TypeVar` for generic functions/classes."

---

### 9.11 Common Built-in Functions Worth Knowing

> **📣 Definition:** _"Python's built-in functions are the everyday toolkit you'll use constantly. `enumerate` gives you index + value while iterating, `zip` pairs up multiple sequences, `sorted` with `key=` handles arbitrary sort criteria, `any`/`all` for boolean reductions, `isinstance` for type checking that respects inheritance, and `getattr`/`hasattr`/`setattr` for dynamic attribute access. Mastering these is what separates someone who 'knows Python syntax' from someone who writes Pythonic code."_

```python
# enumerate — get index + value
for i, val in enumerate(['a', 'b', 'c']):
    print(i, val)              # 0 a, 1 b, 2 c

# zip — pair up sequences
names = ["Tushar", "Aman"]
ages = [30, 25]
for name, age in zip(names, ages):
    print(name, age)

# zip with unpacking
pairs = [(1, 'a'), (2, 'b')]
nums, letters = zip(*pairs)    # nums=(1,2), letters=('a','b')

# any / all
any([False, True, False])      # True (at least one truthy)
all([True, True, False])       # False (not all truthy)

# sorted with key
people = [{"name": "B", "age": 30}, {"name": "A", "age": 25}]
sorted(people, key=lambda p: p["age"])         # sort by age
sorted(people, key=lambda p: (p["age"], p["name"]))  # multi-key

# reversed
list(reversed([1, 2, 3]))      # [3, 2, 1]

# range with step
list(range(10, 0, -2))         # [10, 8, 6, 4, 2]

# isinstance — type check (preferred over `type()`)
isinstance(x, (int, float))    # works for tuples of types

# hasattr / getattr / setattr — dynamic attribute access
if hasattr(obj, "save"):
    obj.save()

method = getattr(obj, "save", default_func)    # safe access
setattr(obj, "x", 10)

# globals() / locals() — namespace access (debugging)

# divmod — quotient + remainder
divmod(17, 5)                  # (3, 2)

# round, abs, pow, max, min, sum
max([1, 2, 3])                 # 3
max([1, 2, 3], key=lambda x: -x)  # 1 (max by negation)

# id — memory address (for `is` debugging)
print(id(x), id(y))
```

📌 **TLDR:** "Master `enumerate`, `zip`, `sorted` with `key`, `any`/`all`, `isinstance`, and `getattr`/`hasattr` — these come up constantly. Always prefer `isinstance(x, T)` over `type(x) == T` because it respects inheritance."

---

### 9.12 Exception Handling Best Practices

> **📣 Definition:** _"Exception handling in Python uses `try/except/else/finally` — `else` runs only on success, `finally` always runs. The golden rules: catch specific exceptions, never bare `except:` (it swallows even `KeyboardInterrupt`), preserve the original cause with `raise X from e`, and always re-raise after logging unless you have a real reason to suppress. Custom exceptions should subclass `Exception`, not `BaseException`. Python idiom favors EAFP — 'Easier to Ask Forgiveness than Permission' — over LBYL ('Look Before You Leap')."_

```python
# Basic structure
try:
    risky_call()
except ValueError as e:
    handle_value_error(e)
except (KeyError, IndexError) as e:           # multiple types in one except
    handle_lookup(e)
except Exception as e:
    log(e)
    raise                                      # re-raise after logging
else:
    # runs only if no exception
    cleanup_success()
finally:
    # ALWAYS runs
    close_resources()

# Custom exceptions
class InsufficientFundsError(Exception):
    def __init__(self, balance, requested):
        super().__init__(f"Balance {balance} < requested {requested}")
        self.balance = balance
        self.requested = requested

try:
    withdraw(...)
except InsufficientFundsError as e:
    print(f"Need {e.requested - e.balance} more")

# Exception chaining (preserve the cause)
try:
    db.query(...)
except DBError as e:
    raise ServiceError("Database failed") from e   # preserves original traceback

# DON'T do this — silent failure
try:
    risky()
except:                # ❌ catches EVERYTHING including KeyboardInterrupt, SystemExit
    pass               # ❌ swallowing exceptions = nightmare to debug

# Better
try:
    risky()
except SpecificError as e:
    log.warning(f"expected failure: {e}")

# Exception groups (Python 3.11+)
try:
    raise ExceptionGroup("multiple failures", [
        ValueError("bad value"),
        KeyError("missing key"),
    ])
except* ValueError as eg:
    print("ValueErrors:", eg.exceptions)
except* KeyError as eg:
    print("KeyErrors:", eg.exceptions)
```

📌 **TLDR:** "Catch specific exceptions, not bare `except:`. Use `try/except/else/finally` — `else` runs only on success, `finally` always. Re-raise with `raise` to preserve traceback. Chain with `raise X from e` to keep the cause. Custom exceptions should subclass `Exception`, not `BaseException`."

---

### 9.13 Monkey Patching

> **📣 Definition:** _"Monkey patching is the practice of dynamically modifying or replacing attributes, methods, or functions of a class or module at runtime — after it's already been defined. It's most commonly used in testing (via `unittest.mock.patch`) to swap out external dependencies like API calls or database queries. In production code, it's used sparingly for hotfixes when you can't modify the source. The risks are real: it makes code fragile, harder to debug, and can silently break when the patched library updates. Use it as a scalpel, not a sledgehammer."_

**Layman:** Imagine a vending machine is broken — instead of opening it up and fixing the internal wiring (modifying the source), you tape a new button on the outside that redirects to the correct slot. It works, but anyone maintaining the machine later will be very confused.

**Technical:**

```python
# === Basic Monkey Patching ===

# Original class
class PaymentGateway:
    def charge(self, amount):
        # Calls real payment API
        return requests.post("https://api.stripe.com/charge", json={"amount": amount})

# Monkey patch at runtime — replace the method
def fake_charge(self, amount):
    return {"status": "success", "amount": amount, "fake": True}

PaymentGateway.charge = fake_charge   # 🐒 patched!

gw = PaymentGateway()
gw.charge(500)   # No real API call — uses fake_charge


# === Monkey Patching a Module ===
import json

# Replace json.dumps globally
_original_dumps = json.dumps
def logging_dumps(obj, **kwargs):
    print(f"[DEBUG] Serializing: {type(obj).__name__}")
    return _original_dumps(obj, **kwargs)

json.dumps = logging_dumps   # every json.dumps call in the entire app now logs


# === The RIGHT Way: unittest.mock.patch (scoped, safe, auto-reverted) ===
from unittest.mock import patch, MagicMock

class TestPayment:
    @patch("myapp.payments.PaymentGateway.charge")
    def test_checkout(self, mock_charge):
        mock_charge.return_value = {"status": "success", "id": "txn_123"}

        result = checkout(amount=500)

        mock_charge.assert_called_once_with(500)
        assert result["status"] == "success"
        # After the test, .charge is automatically restored!

    # Context manager style
    def test_checkout_v2(self):
        with patch("myapp.payments.requests.post") as mock_post:
            mock_post.return_value = MagicMock(
                status_code=200,
                json=lambda: {"status": "ok"}
            )
            result = PaymentGateway().charge(100)
            mock_post.assert_called_once()


# === Monkey Patching for Hotfixes (production — use with extreme caution) ===
# Scenario: third-party library has a bug, fix isn't released yet
import third_party_lib

def patched_buggy_method(self, data):
    # Fixed version of the buggy method
    if data is None:
        return []           # ← the fix: handle None input
    return self._original_process(data)

third_party_lib.Processor._original_process = third_party_lib.Processor.process
third_party_lib.Processor.process = patched_buggy_method
# Document heavily, add a TODO to remove when library updates!


# === gevent/eventlet monkey patching (async I/O) ===
# These libraries monkey-patch the entire stdlib to make blocking I/O async
from gevent import monkey
monkey.patch_all()   # patches socket, ssl, threading, time, etc.
# Now all blocking calls become cooperative — standard library unchanged to callers
```

**When monkey patching is acceptable vs dangerous:**

| ✅ Acceptable                                          | ❌ Dangerous                                    |
| ------------------------------------------------------ | ----------------------------------------------- |
| Unit tests via `unittest.mock.patch`                   | Patching methods in production permanently      |
| Temporary hotfix for 3rd-party bug (documented + TODO) | Patching built-in types (`str`, `int`)          |
| gevent/eventlet stdlib patching (well-understood)      | Patching without preserving the original        |
| Feature flags that swap implementations                | Patching across multiple modules inconsistently |

📌 **TLDR:** "Monkey patching = dynamically replacing methods/attributes at runtime. The safe way is `unittest.mock.patch` (scoped, auto-reverted). In production, use it only for hotfixes on third-party code you can't modify, always preserve the original, and document heavily. gevent/eventlet use it to make blocking I/O async. The risks: fragile code, invisible side effects, breaks on library updates."

---

### 9.14 Closures & the LEGB Scope Rule

> **📣 Definition:** _"A closure is a function that 'remembers' variables from the enclosing scope where it was defined, even after that scope has finished executing. Python resolves variable names using the LEGB rule: Local → Enclosing → Global → Built-in. Closures are the mechanism behind decorators, function factories, and callback patterns. The classic gotcha: closures capture variables by reference, not by value — so a closure over a loop variable sees the variable's final value, not the value at creation time. Fix with default arguments or `functools.partial`."_

**Layman:** A closure is like a backpack that a function carries with it. When the function leaves its home (the enclosing function), it takes the backpack (the enclosed variables) along — and can access those items anywhere, anytime.

**Technical:**

```python
# === LEGB Rule — Variable Lookup Order ===
# L = Local (inside the current function)
# E = Enclosing (inside enclosing function — for nested functions)
# G = Global (module-level)
# B = Built-in (Python's built-in names like len, print, etc.)

x = "global"                    # G — global scope

def outer():
    x = "enclosing"             # E — enclosing scope

    def inner():
        x = "local"             # L — local scope
        print(x)                # "local" — L wins

    inner()

outer()

# Built-in example
# len = 42                     # ← if you do this, you shadow the built-in len()
# len([1,2,3])                 # TypeError! Python found len=42 at G before B


# === Basic Closure ===
def make_multiplier(factor):
    def multiply(x):
        return x * factor       # 'factor' is captured from enclosing scope
    return multiply             # return the inner function (with its backpack!)

double = make_multiplier(2)
triple = make_multiplier(3)

print(double(5))               # 10
print(triple(5))               # 15
print(double.__closure__[0].cell_contents)   # 2 — you can inspect the captured value!


# === Closure for State (alternative to classes for simple cases) ===
def make_counter(start=0):
    count = start
    def increment():
        nonlocal count          # ← required to MODIFY the enclosed variable
        count += 1
        return count
    def get():
        return count            # reading doesn't need nonlocal
    return increment, get

inc, get = make_counter(10)
inc()               # 11
inc()               # 12
print(get())        # 12


# === THE CLASSIC GOTCHA: Late Binding in Loops ===
# ❌ WRONG — all functions see i=4 (the final value)
functions = []
for i in range(5):
    functions.append(lambda: i)

print([f() for f in functions])    # [4, 4, 4, 4, 4]  ← surprise!

# ✅ FIX 1: Default argument captures current value
functions = []
for i in range(5):
    functions.append(lambda i=i: i)    # i=i captures by VALUE at creation time

print([f() for f in functions])    # [0, 1, 2, 3, 4]  ✅

# ✅ FIX 2: functools.partial
from functools import partial
functions = [partial(lambda i: i, i) for i in range(5)]
print([f() for f in functions])    # [0, 1, 2, 3, 4]  ✅


# === How Decorators Use Closures ===
def retry(max_attempts=3):            # ← decorator factory (returns a decorator)
    def decorator(func):              # ← the actual decorator (closure over max_attempts)
        def wrapper(*args, **kwargs): # ← the wrapper (closure over func AND max_attempts)
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"Retry {attempt + 1}/{max_attempts}")
        return wrapper
    return decorator

@retry(max_attempts=5)
def flaky_api_call():
    ...
# Three layers of closures! wrapper closes over func + max_attempts.
```

📌 **TLDR:** "LEGB = Local → Enclosing → Global → Built-in — Python's name resolution order. A closure captures variables from the enclosing scope by _reference_, not value. Use `nonlocal` to modify enclosed variables. The classic trap: closures in loops capture the loop variable's final value — fix with default arguments `lambda i=i: i`. Closures are the engine behind decorators and function factories."

---

### 9.15 First-Class Functions & Higher-Order Functions

> **📣 Definition:** _"In Python, functions are first-class objects — they can be assigned to variables, stored in data structures, passed as arguments, and returned from other functions. A higher-order function is one that takes a function as an argument or returns one. This is the foundation for decorators, callbacks, `sorted(key=...)`, `map`/`filter`, and strategy patterns. Understanding this is what separates 'I know Python syntax' from 'I think in Python'."_

```python
# === Functions are objects — they have attributes ===
def greet(name):
    """Say hello."""
    return f"Hello, {name}!"

print(type(greet))              # <class 'function'>
print(greet.__name__)           # 'greet'
print(greet.__doc__)            # 'Say hello.'

# Assign to a variable
say_hi = greet                  # NOT calling it — assigning the function object
print(say_hi("Tushar"))         # "Hello, Tushar!"

# Store in data structures
operations = {
    "add": lambda a, b: a + b,
    "sub": lambda a, b: a - b,
    "mul": lambda a, b: a * b,
}
print(operations["add"](3, 4))  # 7


# === Higher-Order Functions — accept or return functions ===

# 1. Function as argument (callback pattern)
def apply_operation(func, x, y):
    return func(x, y)

apply_operation(lambda a, b: a ** b, 2, 10)   # 1024

# 2. Function as return value (factory pattern)
def make_validator(min_val, max_val):
    def validate(x):
        return min_val <= x <= max_val
    return validate

is_valid_age = make_validator(0, 150)
is_valid_percentage = make_validator(0, 100)

print(is_valid_age(25))          # True
print(is_valid_percentage(105))  # False

# 3. Strategy pattern using first-class functions
def process_data(data, strategy):
    return strategy(data)

def sort_strategy(data):     return sorted(data)
def reverse_strategy(data):  return list(reversed(data))
def unique_strategy(data):   return list(set(data))

process_data([3, 1, 2, 1], sort_strategy)      # [1, 1, 2, 3]
process_data([3, 1, 2, 1], unique_strategy)     # [1, 2, 3]

# 4. Built-in higher-order functions
sorted(["banana", "apple", "cherry"], key=len)              # by length
sorted(users, key=lambda u: (u.role, u.name))               # multi-key
min(products, key=lambda p: p.price)                        # cheapest
```

📌 **TLDR:** "Functions are first-class objects in Python — assignable, passable, returnable. Higher-order functions take or return functions. This enables decorators, callbacks, strategy patterns, and `sorted(key=...)`. If you understand this, decorators and closures become obvious."

---

### 9.16 `classmethod` vs `staticmethod` vs Instance Method

> **📣 Definition:** _"Python has three kinds of methods. **Instance methods** receive `self` (the instance) and operate on instance data. **`@classmethod`** receives `cls` (the class itself) and is used for alternative constructors or class-level operations. **`@staticmethod`** receives neither — it's just a regular function namespaced inside the class for logical grouping. The key interview distinction: `classmethod` respects inheritance (returns the correct subclass), `staticmethod` doesn't know about the class at all."_

```python
class Pizza:
    base_price = 10

    def __init__(self, size, toppings):
        self.size = size
        self.toppings = toppings

    # Instance method — operates on a specific pizza
    def price(self):
        return self.base_price + len(self.toppings) * 2 + self.size

    # Class method — alternative constructor
    @classmethod
    def margherita(cls, size=12):
        """Factory method — creates a specific kind of pizza."""
        return cls(size, ["mozzarella", "tomato", "basil"])
        #      ^^^ uses cls, NOT Pizza — so subclasses work!

    @classmethod
    def from_string(cls, pizza_string):
        """Parse 'size:topping1,topping2' format."""
        size, toppings = pizza_string.split(":")
        return cls(int(size), toppings.split(","))

    # Static method — utility, no access to cls or self
    @staticmethod
    def is_valid_topping(topping):
        """Doesn't need the instance or class — pure logic."""
        valid = {"mozzarella", "tomato", "basil", "pepperoni", "mushroom"}
        return topping.lower() in valid


# Usage
p = Pizza.margherita()                            # classmethod as factory
p2 = Pizza.from_string("14:pepperoni,mushroom")   # classmethod as parser
print(Pizza.is_valid_topping("pineapple"))         # False — staticmethod


# === Why classmethod matters for inheritance ===
class VeganPizza(Pizza):
    base_price = 12   # more expensive base

v = VeganPizza.margherita(size=10)
print(type(v))       # <class 'VeganPizza'> ← cls was VeganPizza, NOT Pizza!
print(v.price())     # uses VeganPizza.base_price = 12

# If margherita() used Pizza(...) instead of cls(...), it would always return Pizza.
# This is the #1 reason classmethod exists — inheritance-safe constructors.


# === When to use which ===
# Instance method  → needs self (instance data)          → 95% of methods
# @classmethod     → needs cls (factory, class-level)    → alternative constructors
# @staticmethod    → needs neither                       → utility functions
```

📌 **TLDR:** "Instance method gets `self`, classmethod gets `cls`, staticmethod gets neither. Use classmethod for alternative constructors (`from_string`, `from_dict`) — it respects inheritance via `cls(...)`. Use staticmethod for pure utility functions that logically belong to the class but don't need instance or class data."

---

### 9.17 `__new__` vs `__init__` — Object Creation vs Initialization

> **📣 Definition:** _"`__new__` is responsible for creating the instance (allocating memory, returning the new object), while `__init__` initializes it (setting attributes on the already-created object). For mutable types, you rarely override `__new__` — just use `__init__`. For immutable types (`str`, `int`, `tuple`, `frozenset`), you MUST override `__new__` because by the time `__init__` runs, the object is already frozen. `__new__` is also the hook for implementing singletons and custom metaclasses."_

```python
# === Normal flow: __new__ creates, __init__ initializes ===
class MyClass:
    def __new__(cls, *args, **kwargs):
        print(f"1. __new__ called — creating instance of {cls.__name__}")
        instance = super().__new__(cls)       # actually allocate memory
        return instance                        # MUST return the instance!

    def __init__(self, value):
        print(f"2. __init__ called — initializing with {value}")
        self.value = value                     # set attributes on existing instance

obj = MyClass(42)
# Output:
# 1. __new__ called — creating instance of MyClass
# 2. __init__ called — initializing with 42


# === Singleton Pattern using __new__ ===
class DatabaseConnection:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance                   # always return the SAME instance

    def __init__(self, url="localhost:5432"):
        self.url = url

a = DatabaseConnection("db1.example.com")
b = DatabaseConnection("db2.example.com")
print(a is b)         # True — same object!
print(a.url)          # "db2.example.com" — __init__ ran twice on same instance!


# === Immutable types — MUST use __new__ ===
class UpperStr(str):
    """A string subclass that is always uppercase."""
    def __new__(cls, value):
        # str is immutable — can't change it after creation
        # so we must uppercase BEFORE the object is created
        instance = super().__new__(cls, value.upper())
        return instance

    # __init__ is too late — the str value is already frozen!

s = UpperStr("hello")
print(s)              # "HELLO"
print(isinstance(s, str))   # True


class PositiveInt(int):
    """An int that is always positive."""
    def __new__(cls, value):
        return super().__new__(cls, abs(value))

print(PositiveInt(-42))   # 42


# === __init_subclass__ — lightweight hook for subclass creation (Python 3.6+) ===
class Plugin:
    _registry = {}

    def __init_subclass__(cls, plugin_name=None, **kwargs):
        super().__init_subclass__(**kwargs)
        name = plugin_name or cls.__name__.lower()
        Plugin._registry[name] = cls
        print(f"Registered plugin: {name}")

class EmailPlugin(Plugin, plugin_name="email"):
    pass

class SlackPlugin(Plugin, plugin_name="slack"):
    pass

print(Plugin._registry)
# {'email': <class 'EmailPlugin'>, 'slack': <class 'SlackPlugin'>}
# No metaclass needed! __init_subclass__ is the modern, simpler way.
```

| Aspect               | `__new__`                                | `__init__`                                               |
| -------------------- | ---------------------------------------- | -------------------------------------------------------- |
| **Purpose**          | Create the instance (allocate)           | Initialize the instance (configure)                      |
| **Receives**         | `cls` (the class)                        | `self` (the instance)                                    |
| **Must return**      | The new instance                         | Nothing (`None`)                                         |
| **When to override** | Immutable types, singletons, metaclasses | 99% of the time — this is your constructor               |
| **Called**           | Before `__init__`                        | After `__new__` (only if `__new__` returns correct type) |

📌 **TLDR:** "`__new__` creates the object (allocates memory, returns it). `__init__` initializes the created object (sets attributes). Override `__new__` only for immutable types (str, int, tuple subclasses), singletons, or metaclasses. For everything else, use `__init__`. Use `__init_subclass__` for subclass registration hooks — it's the modern alternative to metaclasses for most use cases."

---

### 9.18 Metaclasses — The "Class of a Class"

> **📣 Definition:** _"A metaclass is what creates class objects — it's the 'class of a class'. Just as a class defines how instances behave, a metaclass defines how classes behave. The default metaclass is `type` — when you write `class Foo:`, Python calls `type('Foo', bases, namespace)` behind the scenes. Custom metaclasses let you intercept class creation to enforce coding standards, auto-register classes, modify attributes, or build frameworks like Django's ORM (where `class User(Model):` magically creates database tables). For most use cases in 2026, prefer `__init_subclass__` or class decorators over custom metaclasses — they're simpler and cover 90% of needs."_

**Layman:** If a class is a blueprint for objects, a metaclass is a blueprint for blueprints. It controls HOW classes themselves get built.

**Technical:**

```python
# === Everything is an object — including classes ===
class Dog:
    pass

print(type(Dog))          # <class 'type'>  — Dog is an INSTANCE of type
print(type(type))         # <class 'type'>  — type is its own metaclass (bootstrapped)

# You can create classes dynamically with type()
Cat = type("Cat", (object,), {"sound": "meow", "speak": lambda self: self.sound})
c = Cat()
print(c.speak())          # "meow"


# === Custom Metaclass ===
class ValidatedMeta(type):
    """Metaclass that enforces all subclasses must define a 'validate' method."""

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)

        # Skip validation for the base class itself
        if bases:  # only check subclasses, not the base
            if "validate" not in namespace:
                raise TypeError(
                    f"Class {name} must implement a 'validate' method"
                )
        return cls

class Serializer(metaclass=ValidatedMeta):
    """Base class — subclasses MUST implement validate()."""
    pass

class UserSerializer(Serializer):
    def validate(self, data):       # ✅ has validate
        return bool(data.get("email"))

# class BadSerializer(Serializer):  # ❌ TypeError at CLASS DEFINITION time!
#     pass                          # Missing validate method


# === Auto-Registration Pattern (Django-style) ===
class ModelMeta(type):
    registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # don't register the base Model class itself
            ModelMeta.registry[name] = cls
        return cls

class Model(metaclass=ModelMeta):
    pass

class User(Model):
    pass

class Product(Model):
    pass

print(ModelMeta.registry)
# {'User': <class 'User'>, 'Product': <class 'Product'>}


# === Class Creation Order (what happens when you write `class Foo:`) ===
# 1. __prepare__(mcs, name, bases)    → returns the namespace dict (can be OrderedDict, etc.)
# 2. Class body executes              → fills the namespace
# 3. __new__(mcs, name, bases, ns)    → creates the class object
# 4. __init__(cls, name, bases, ns)   → initializes the class object
# 5. __init_subclass__()              → notifies parent class (Python 3.6+)


# === When to use metaclass vs alternatives ===
# __init_subclass__  → simple registration, validation     → PREFER THIS
# Class decorator    → modify a class after creation        → simpler than metaclass
# Metaclass          → control class creation itself        → frameworks, ORMs, ABCs
```

📌 **TLDR:** "A metaclass is the 'class of a class' — it controls how classes are created. The default metaclass is `type`. Custom metaclasses let you enforce interfaces, auto-register classes, or modify class attributes at definition time. Django's ORM uses metaclasses to turn `class User(Model):` into database tables. For most use cases, prefer `__init_subclass__` (simpler) or class decorators. Reach for metaclasses only when you need to control the class creation process itself."

---

### 9.19 Descriptors — The Machinery Behind `@property`

> **📣 Definition:** _"A descriptor is any object that defines `__get__`, `__set__`, or `__delete__`. Descriptors are the hidden mechanism behind `@property`, `@classmethod`, `@staticmethod`, and even regular method binding. A **data descriptor** (defines `__set__` or `__delete__`) takes priority over an instance's `__dict__`; a **non-data descriptor** (only `__get__`) does not. Understanding descriptors means understanding how Python attribute access actually works under the hood — which is a favorite senior-level interview topic."_

**Layman:** A descriptor is like a smart mailbox. Instead of just storing mail (a regular attribute), it has custom logic for what happens when you put something in, take something out, or check what's inside.

**Technical:**

```python
# === Basic Descriptor — Type-Checked Attribute ===
class TypeChecked:
    """Descriptor that enforces type on assignment."""
    def __init__(self, expected_type):
        self.expected_type = expected_type
        self.name = None                   # set by __set_name__

    def __set_name__(self, owner, name):
        """Called automatically when the descriptor is assigned to a class attribute."""
        self.name = name                   # Python 3.6+ — no need to pass name manually

    def __get__(self, instance, owner):
        if instance is None:
            return self                    # class-level access returns the descriptor itself
        return instance.__dict__.get(self.name)

    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        instance.__dict__[self.name] = value

    def __delete__(self, instance):
        del instance.__dict__[self.name]


class User:
    name = TypeChecked(str)                # descriptor instances as CLASS attributes
    age = TypeChecked(int)

    def __init__(self, name, age):
        self.name = name                   # calls TypeChecked.__set__
        self.age = age

u = User("Tushar", 30)                     # ✅
u.name = "Aman"                            # ✅ calls __set__ with type check
# u.age = "thirty"                         # ❌ TypeError: age must be int, got str


# === How @property is implemented (simplified) ===
# @property is just a built-in descriptor!
class property_from_scratch:
    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, instance, owner):
        if instance is None:
            return self
        return self.fget(instance)

    def __set__(self, instance, value):
        if self.fset is None:
            raise AttributeError("read-only")
        self.fset(instance, value)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)


# === Data vs Non-Data Descriptors ===
# Data descriptor:     defines __set__ or __delete__  → WINS over instance __dict__
# Non-data descriptor: defines only __get__           → instance __dict__ wins

class NonDataDesc:
    def __get__(self, instance, owner):
        return "from descriptor"

class DataDesc:
    def __get__(self, instance, owner):
        return "from descriptor"
    def __set__(self, instance, value):
        instance.__dict__["attr"] = value

class MyClass:
    non_data = NonDataDesc()
    data = DataDesc()

obj = MyClass()
obj.__dict__["non_data"] = "from instance"
obj.__dict__["data"] = "from instance"

print(obj.non_data)    # "from instance"     ← instance dict WINS for non-data
print(obj.data)        # "from descriptor"   ← data descriptor WINS over instance dict


# === Attribute Lookup Order (important for interviews!) ===
# 1. Data descriptors on the class (and its MRO)
# 2. Instance __dict__
# 3. Non-data descriptors on the class (and its MRO)
# 4. __getattr__ (if defined) — fallback
```

📌 **TLDR:** "Descriptors define `__get__`/`__set__`/`__delete__` and control attribute access. They're the mechanism behind `@property`, `@classmethod`, `@staticmethod`, and method binding. Data descriptors (with `__set__`) override instance `__dict__`; non-data descriptors don't. The attribute lookup order: data descriptors → instance dict → non-data descriptors → `__getattr__`."

---

### 9.20 `dataclass` vs `namedtuple` — Choosing the Right Data Container

> **📣 Definition:** _"`dataclass` (3.7+) and `namedtuple` both reduce boilerplate for data-holding classes, but they serve different needs. `namedtuple` creates an immutable, tuple-backed container — lightweight, hashable, and unpacking-friendly but inflexible. `dataclass` creates a regular mutable class with auto-generated `__init__`, `__repr__`, `__eq__` — it supports defaults, type hints, inheritance, and can be made immutable with `frozen=True`. In 2026, `dataclass` is the default choice for most data containers; `namedtuple` is for when you need tuple semantics (unpacking, immutability by default) or interop with code expecting tuples."_

```python
# === namedtuple — immutable, tuple-backed ===
from collections import namedtuple

Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)

print(p.x, p.y)           # 3 4    — named access
print(p[0], p[1])          # 3 4    — index access (it IS a tuple)
x, y = p                   # unpacking works
print(p)                   # Point(x=3, y=4)

# p.x = 5                 # ❌ AttributeError — immutable!
p2 = p._replace(x=5)      # creates a new Point with x=5

# namedtuple with defaults (Python 3.6.1+)
Config = namedtuple("Config", ["host", "port", "debug"], defaults=["localhost", 8080, False])
c = Config()               # Config(host='localhost', port=8080, debug=False)

# Typed version (Python 3.6+)
from typing import NamedTuple

class Coordinate(NamedTuple):
    x: float
    y: float
    label: str = "origin"


# === dataclass — mutable, flexible, modern ===
from dataclasses import dataclass, field, asdict, astuple

@dataclass
class User:
    name: str
    email: str
    age: int = 0                              # default value
    tags: list[str] = field(default_factory=list)   # mutable default — SAFE!
    _internal: str = field(repr=False, compare=False, default="")

    def __post_init__(self):
        """Runs after __init__ — for validation or computed fields."""
        self.email = self.email.lower()

u = User("Tushar", "Tushar@Example.com", 30)
print(u)               # User(name='Tushar', email='tushar@example.com', age=30, tags=[])
print(asdict(u))       # {'name': 'Tushar', 'email': 'tushar@example.com', ...}

# Equality — auto-generated __eq__
u2 = User("Tushar", "tushar@example.com", 30)
print(u == u2)         # True


# === Frozen dataclass — immutable (like namedtuple but more features) ===
@dataclass(frozen=True)
class Money:
    amount: int
    currency: str = "INR"

m = Money(500)
# m.amount = 600       # ❌ FrozenInstanceError
print(hash(m))          # ✅ hashable — can use as dict key / in set


# === Dataclass with __slots__ (Python 3.10+) — memory efficient ===
@dataclass(slots=True)
class SensorReading:
    timestamp: float
    value: float
    sensor_id: str
# Uses __slots__ instead of __dict__ — ~20-30% less memory per instance


# === Ordering (for sorting) ===
@dataclass(order=True)
class Priority:
    level: int
    label: str = field(compare=False)    # excluded from ordering

tasks = [Priority(3, "low"), Priority(1, "critical"), Priority(2, "medium")]
sorted(tasks)       # sorted by level only
```

**Head-to-head comparison:**

| Feature              | `namedtuple`                  | `dataclass`                        |
| -------------------- | ----------------------------- | ---------------------------------- |
| **Mutable**          | ❌ No (immutable)             | ✅ Yes (unless `frozen=True`)      |
| **Indexing**         | ✅ Yes (`p[0]`)               | ❌ No                              |
| **Unpacking**        | ✅ Yes (`x, y = p`)           | ❌ Not by default                  |
| **Inheritance**      | ⚠️ Limited                    | ✅ Full support                    |
| **Defaults**         | ✅ (3.6.1+)                   | ✅                                 |
| **Type hints**       | ✅ via `NamedTuple`           | ✅ Native                          |
| **Mutable defaults** | ❌ Manual                     | ✅ `field(default_factory=...)`    |
| **`__post_init__`**  | ❌                            | ✅                                 |
| **`__slots__`**      | ❌ Manual                     | ✅ (3.10+, `slots=True`)           |
| **Hashable**         | ✅ Always                     | Only with `frozen=True`            |
| **Memory**           | Very low (tuple)              | Normal (dict-based) or low (slots) |
| **Best for**         | Simple records, tuple interop | Complex data models, DTOs, configs |

📌 **TLDR:** "`dataclass` is the default choice for data containers in modern Python — mutable, type-hinted, supports defaults, inheritance, and `__post_init__`. Use `namedtuple` when you need tuple semantics (indexing, unpacking, immutability) or minimal memory. Use `frozen=True` + `slots=True` on dataclass for an immutable, memory-efficient record that's essentially a better namedtuple."

---

### 9.21 Weak References (`weakref`) — Avoiding Memory Leaks

> **📣 Definition:** _"A weak reference points to an object without increasing its reference count — meaning the object can still be garbage collected even if weak references to it exist. This is critical for caches, observer patterns, and parent-child relationships where you don't want the reference itself to keep the object alive. Python's `weakref` module provides `weakref.ref()` for individual references, `WeakValueDictionary` for cache-like structures, and `WeakSet` for collections. When the referenced object is collected, the weak reference returns `None` (or triggers a callback)."_

**Layman:** A strong reference is like holding someone's hand — they can't leave as long as you're holding on. A weak reference is like knowing someone's phone number — they can still leave the party (get garbage collected), and when you call, the number is disconnected (`None`).

**Technical:**

```python
import weakref

# === Basic weak reference ===
class ExpensiveObject:
    def __init__(self, name):
        self.name = name
    def __repr__(self):
        return f"ExpensiveObject({self.name})"

obj = ExpensiveObject("heavy")
weak = weakref.ref(obj)              # weak reference — does NOT increase refcount

print(weak())                         # ExpensiveObject(heavy) — dereference with ()
print(weak() is obj)                  # True — same object

del obj                               # remove the only strong reference
print(weak())                         # None — object was garbage collected!


# === Callback on collection ===
def on_finalize(ref):
    print(f"Object was garbage collected! Ref: {ref}")

obj = ExpensiveObject("tracked")
weak = weakref.ref(obj, on_finalize)
del obj                               # prints: "Object was garbage collected!"


# === WeakValueDictionary — cache that doesn't prevent GC ===
class ImageCache:
    def __init__(self):
        self._cache = weakref.WeakValueDictionary()

    def get_image(self, path):
        img = self._cache.get(path)
        if img is not None:
            return img
        img = self._load_image(path)     # expensive operation
        self._cache[path] = img          # cached, but won't prevent GC
        return img

    def _load_image(self, path):
        return ExpensiveObject(f"image:{path}")

# When no other code holds a reference to an image,
# it gets garbage collected even though it's in the cache.
# The cache entry silently disappears. No memory leak!


# === WeakSet — track instances without preventing GC ===
class Sensor:
    _instances = weakref.WeakSet()      # track all live sensors

    def __init__(self, sensor_id):
        self.sensor_id = sensor_id
        Sensor._instances.add(self)

    @classmethod
    def get_all_active(cls):
        return list(cls._instances)     # only returns sensors still alive

s1 = Sensor("temp_1")
s2 = Sensor("temp_2")
print(len(Sensor.get_all_active()))     # 2
del s1
print(len(Sensor.get_all_active()))     # 1 — s1 was collected


# === finalize — destructor-like cleanup (safer than __del__) ===
import weakref

class TempFile:
    def __init__(self, path):
        self.path = path
        # Register cleanup — runs when object is GC'd, even if __del__ fails
        self._finalizer = weakref.finalize(self, self._cleanup, path)

    @staticmethod
    def _cleanup(path):
        print(f"Cleaning up temp file: {path}")
        # os.remove(path)

t = TempFile("/tmp/data.csv")
del t                                    # prints: Cleaning up temp file: /tmp/data.csv


# === What CANNOT have weak references ===
# Most built-in types: int, str, tuple, list, dict, set
# weakref.ref(42)       # ❌ TypeError
# weakref.ref("hello")  # ❌ TypeError
# Only class instances and a few built-in types (like set) support weakrefs
```

📌 **TLDR:** "Weak references (`weakref.ref`) point to objects without preventing garbage collection. Use `WeakValueDictionary` for caches that don't leak memory, `WeakSet` for tracking live instances, and `weakref.finalize` for safe cleanup. Objects only die when all _strong_ references are gone — weak refs just become `None`. Most built-in types (int, str, list) don't support weak references."

---

### 9.22 String Interning & Memory Optimization Tricks

> **📣 Definition:** _"String interning is a CPython optimization where certain strings (identifiers, small strings) are stored only once in memory. When two variables point to the same interned string, `is` returns `True` because they're literally the same object. This speeds up dict lookups and attribute access (identity check instead of character-by-character comparison). Python automatically interns strings that look like identifiers; you can manually intern with `sys.intern()`. This is an implementation detail — never rely on it for logic. Related optimizations: small integer caching (-5 to 256), `__slots__` for memory, and generators for lazy evaluation."_

```python
import sys

# === String Interning — automatic for identifier-like strings ===
a = "hello"
b = "hello"
print(a is b)              # True — Python interned this string (looks like an identifier)

a = "hello world"
b = "hello world"
print(a is b)              # False (or True — depends on context, NEVER rely on this!)
# Strings with spaces/special chars are NOT automatically interned

# Manual interning
a = sys.intern("hello world")
b = sys.intern("hello world")
print(a is b)              # True — now they're the same object, guaranteed

# WHY intern? — Performance for repeated lookups
# dict key lookup: if interned, Python checks identity (O(1)) before equality (O(n))
# This is why Python interns all attribute names and variable names internally


# === Small Integer Caching (-5 to 256) ===
a = 256
b = 256
print(a is b)              # True — cached

a = 257
b = 257
print(a is b)              # False (usually) — not cached
# NEVER use `is` for integer comparison — always use `==`


# === Memory Optimization Techniques ===

# 1. __slots__ — replace per-instance __dict__ with fixed slots
class PointDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y
# Each instance has a __dict__ (~104 bytes overhead)

class PointSlots:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y
# No __dict__ — saves ~40-60% memory per instance
# Trade-off: can't add arbitrary attributes, no __dict__, limited with multiple inheritance

# 2. Generators instead of lists for large data
# ❌ Creates entire list in memory
total = sum([x**2 for x in range(10**7)])     # ~80MB list
# ✅ Generator — constant memory
total = sum(x**2 for x in range(10**7))       # ~0 bytes extra

# 3. array.array for homogeneous numeric data
from array import array
# list of ints: each int is a Python object (~28 bytes each)
nums_list = list(range(1_000_000))              # ~8MB
# array of ints: raw C integers (~4-8 bytes each)
nums_array = array('i', range(1_000_000))       # ~4MB

# 4. sys.getsizeof() for debugging memory
import sys
print(sys.getsizeof([]))           # 56 bytes (empty list overhead)
print(sys.getsizeof({}))           # 64 bytes (empty dict overhead)
print(sys.getsizeof(""))           # 49 bytes (empty string overhead)
print(sys.getsizeof(0))            # 28 bytes (int)

# 5. __sizeof__() vs sys.getsizeof()
# sys.getsizeof() includes GC overhead; __sizeof__() is just the object
```

📌 **TLDR:** "String interning stores identical strings as one object — Python does it automatically for identifier-like strings, or manually via `sys.intern()`. Small ints (-5 to 256) are also cached. Never use `is` for value comparison — always `==`. For memory optimization: use `__slots__` to eliminate per-instance `__dict__`, generators for lazy evaluation, and `array.array` for homogeneous numeric data."

---

### 9.23 Quick-Reference Topics (One-liner each)

> **📣 Definition:** _"This is a grab-bag of important small concepts: `if __name__ == '__main__':` (run-vs-import detection), `__slots__` (memory optimization), `@dataclass` (auto-generated `__init__`/`__eq__`), the walrus operator `:=`, f-strings with `=` for debugging (`f'{x=}'`), `global` vs `nonlocal` scope modifiers, and Python's truthy/falsy rules. Each comes up enough to know but doesn't need its own deep dive."_

These come up but don't need their own deep dive:

- **`if __name__ == "__main__":`** — runs only when the file is executed directly, not when imported.
- **`pass`** — null statement; placeholder for empty blocks.
- **`global` / `nonlocal`** — modify outer-scope variables (`global` for module level, `nonlocal` for enclosing function scope).
- **`with` statement / `contextlib`** — see [Section 1](#1-python-context-managers-with-statement).
- **`__slots__`** — restrict allowed attributes; saves memory for many instances. `class Point: __slots__ = ('x', 'y')`.
- **`@dataclass`** — auto-generates `__init__`, `__repr__`, `__eq__`. Use for simple data containers.
- **Walrus operator `:=`** (3.8+) — assignment within an expression: `if (n := len(data)) > 100: print(n)`.
- **f-strings (3.6+)** — `f"name={name}, age={age}"`. Supports `=` for debug: `f"{name=}"` → `"name='Tushar'"`.
- **`try/except` is cheap, `if` is cheap too** — Python idiom: EAFP ("Easier to Ask Forgiveness than Permission") often beats LBYL ("Look Before You Leap").
- **Default parameters** — evaluated ONCE at function definition (mutable default trap!).
- **Tuple unpacking** — `a, b = 1, 2`; `first, *rest = [1, 2, 3, 4]` → `first=1, rest=[2,3,4]`.
- **Set operations** — `&` (intersect), `|` (union), `-` (diff), `^` (symmetric diff).
- **Truthy/Falsy** — empty containers (`[]`, `{}`, `""`, `()`), `0`, `0.0`, `None`, `False` are all falsy.

---

### 9.24 Python Data Structure Methods — Complete Cheat Sheet

> **📣 Definition:** _"A quick-reference of every method on Python's built-in and standard-library data structures. Interviewers often test whether you know the right method for the job — `setdefault` vs `get`, `discard` vs `remove`, `appendleft` on deque, etc. This section is your lookup table."_

#### `list` — Ordered, Mutable Sequence

| Method                 | What it does                               | Returns    | Example                          |
| ---------------------- | ------------------------------------------ | ---------- | -------------------------------- |
| `append(x)`            | Add `x` to the end                         | `None`     | `[1,2].append(3)` → `[1,2,3]`    |
| `extend(iter)`         | Append all items from iterable             | `None`     | `[1].extend([2,3])` → `[1,2,3]`  |
| `insert(i, x)`         | Insert `x` at index `i`                    | `None`     | `[1,3].insert(1, 2)` → `[1,2,3]` |
| `remove(x)`            | Remove first occurrence of `x`             | `None`     | `[1,2,3].remove(2)` → `[1,3]`    |
| `pop(i=-1)`            | Remove & return item at index `i`          | The item   | `[1,2,3].pop()` → `3`            |
| `clear()`              | Remove all items                           | `None`     | `[1,2].clear()` → `[]`           |
| `index(x, start, end)` | Index of first `x` (ValueError if missing) | `int`      | `[1,2,3].index(2)` → `1`         |
| `count(x)`             | Count occurrences of `x`                   | `int`      | `[1,2,2,3].count(2)` → `2`       |
| `sort(key=, reverse=)` | Sort **in place**                          | `None`     | `[3,1,2].sort()` → `[1,2,3]`     |
| `reverse()`            | Reverse **in place**                       | `None`     | `[1,2,3].reverse()` → `[3,2,1]`  |
| `copy()`               | Shallow copy                               | New `list` | `[1,2].copy()` → `[1,2]`         |

> **Gotchas:** `sort()` returns `None` (in-place); use `sorted(lst)` for a new list. `remove()` raises `ValueError` if missing. `append` vs `extend`: `[1].append([2,3])` → `[1,[2,3]]` but `[1].extend([2,3])` → `[1,2,3]`.

---

#### `dict` — Key-Value Mapping (insertion-ordered since 3.7)

| Method                   | What it does                             | Returns          | Example                   |
| ------------------------ | ---------------------------------------- | ---------------- | ------------------------- |
| `get(k, default=None)`   | Get value for key, no KeyError           | Value or default | `d.get("x", 0)` → `0`     |
| `setdefault(k, default)` | Get value; if missing, set it to default | Value            | `d.setdefault("x", [])`   |
| `update(other)`          | Merge another dict/iterable of pairs     | `None`           | `d.update({"a": 1})`      |
| `pop(k, default)`        | Remove key & return value                | Value            | `d.pop("x", None)`        |
| `popitem()`              | Remove & return last inserted `(k, v)`   | `tuple`          | LIFO order (3.7+)         |
| `keys()`                 | View of keys                             | `dict_keys`      | `for k in d.keys()`       |
| `values()`               | View of values                           | `dict_values`    | `list(d.values())`        |
| `items()`                | View of `(k, v)` pairs                   | `dict_items`     | `for k,v in d.items()`    |
| `clear()`                | Remove all items                         | `None`           | `d.clear()`               |
| `copy()`                 | Shallow copy                             | New `dict`       | `d.copy()`                |
| `fromkeys(iter, val)`    | Create dict with keys from iter          | New `dict`       | `dict.fromkeys("abc", 0)` |
| `d \| other` (3.9+)      | Merge dicts (new dict)                   | New `dict`       | `{1:2} \| {3:4}`          |
| `d \|= other` (3.9+)     | Merge in place                           | `None`           | `d \|= {"a": 1}`          |

> **Gotchas:** `setdefault` is atomic get-or-set — great for building `dict[list]` without checking. `d[k]` raises `KeyError`; `d.get(k)` returns `None`. Dict views are live — they reflect changes to the dict.

---

#### `set` — Unordered, Unique Elements (Hashable items only)

| Method                              | What it does                         | Returns   | Example                          |
| ----------------------------------- | ------------------------------------ | --------- | -------------------------------- |
| `add(x)`                            | Add element                          | `None`    | `{1,2}.add(3)` → `{1,2,3}`       |
| `remove(x)`                         | Remove `x` (**KeyError** if missing) | `None`    | `{1,2}.remove(1)`                |
| `discard(x)`                        | Remove `x` (no error if missing)     | `None`    | `{1,2}.discard(99)` → OK         |
| `pop()`                             | Remove & return arbitrary element    | Element   | `{1,2}.pop()`                    |
| `clear()`                           | Remove all                           | `None`    | `s.clear()`                      |
| `union(other)` / `\|`               | All elements from both               | New `set` | `{1,2} \| {2,3}` → `{1,2,3}`     |
| `intersection(other)` / `&`         | Common elements                      | New `set` | `{1,2} & {2,3}` → `{2}`          |
| `difference(other)` / `-`           | In self but not other                | New `set` | `{1,2} - {2,3}` → `{1}`          |
| `symmetric_difference(other)` / `^` | In one but not both                  | New `set` | `{1,2} ^ {2,3}` → `{1,3}`        |
| `issubset(other)` / `<=`            | All elements in other?               | `bool`    | `{1} <= {1,2}` → `True`          |
| `issuperset(other)` / `>=`          | Contains all of other?               | `bool`    | `{1,2} >= {1}` → `True`          |
| `isdisjoint(other)`                 | No common elements?                  | `bool`    | `{1,2}.isdisjoint({3})` → `True` |
| `update(other)` / `\|=`             | Add all from other (in place)        | `None`    | `s.update({3,4})`                |
| `intersection_update` / `&=`        | Keep only common (in place)          | `None`    | `s &= {1,2}`                     |
| `difference_update` / `-=`          | Remove all in other (in place)       | `None`    | `s -= {2}`                       |
| `copy()`                            | Shallow copy                         | New `set` | `s.copy()`                       |

> **Gotchas:** `remove` vs `discard` — `remove` raises `KeyError`, `discard` doesn't. `frozenset` has the same read methods but no mutating methods (`add`, `remove`, etc.) — it's hashable and can be used as a dict key.

---

#### `tuple` — Ordered, Immutable Sequence

| Method                 | What it does             | Returns | Example                    |
| ---------------------- | ------------------------ | ------- | -------------------------- |
| `count(x)`             | Count occurrences of `x` | `int`   | `(1,2,2,3).count(2)` → `2` |
| `index(x, start, end)` | Index of first `x`       | `int`   | `(1,2,3).index(3)` → `2`   |

> **That's it** — tuples are immutable, so they only have 2 methods. Everything else is via operators: `+` (concat), `*` (repeat), `in` (membership), slicing, unpacking (`a, b = t`).

**"How do you add/remove from a tuple?"** — You don't. You create a **new** tuple:

```python
t = (1, 2, 3)

# "Add" — concatenation (creates new tuple)
t2 = t + (4,)               # (1, 2, 3, 4)    — note the trailing comma!
t3 = (0,) + t               # (0, 1, 2, 3)    — prepend

# "Remove" — slicing (creates new tuple)
t4 = t[:1] + t[2:]          # (1, 3)           — removed index 1
t5 = tuple(x for x in t if x != 2)  # (1, 3)  — remove by value

# "Modify" — rebuild
t6 = t[:1] + (99,) + t[2:]  # (1, 99, 3)      — replaced index 1

# Convert to list, mutate, convert back (when you need many changes)
lst = list(t)
lst.append(4)
lst.remove(2)
t7 = tuple(lst)              # (1, 3, 4)
```

> **Interview point:** If you need frequent add/remove, use a `list`. Tuples are for fixed-structure data (coordinates, DB rows, dict keys, function return values).

---

#### `str` — Immutable Text Sequence

| Method                          | Category   | What it does                       | Example                                      |
| ------------------------------- | ---------- | ---------------------------------- | -------------------------------------------- |
| `split(sep, maxsplit)`          | Splitting  | Split by separator                 | `"a,b,c".split(",")` → `['a','b','c']`       |
| `rsplit(sep, maxsplit)`         | Splitting  | Split from right                   | `"a.b.c".rsplit(".",1)` → `['a.b','c']`      |
| `splitlines()`                  | Splitting  | Split by line breaks               | `"a\nb".splitlines()` → `['a','b']`          |
| `partition(sep)`                | Splitting  | Split into 3-tuple at first sep    | `"a:b:c".partition(":")` → `('a',':','b:c')` |
| `join(iter)`                    | Joining    | Join iterable with separator       | `",".join(["a","b"])` → `"a,b"`              |
| `strip(chars)`                  | Trimming   | Remove leading/trailing chars      | `" hi ".strip()` → `"hi"`                    |
| `lstrip(chars)`                 | Trimming   | Remove leading chars               | `"  hi".lstrip()` → `"hi"`                   |
| `rstrip(chars)`                 | Trimming   | Remove trailing chars              | `"hi  ".rstrip()` → `"hi"`                   |
| `replace(old, new, count)`      | Replacing  | Replace occurrences                | `"aab".replace("a","x")` → `"xxb"`           |
| `upper()`                       | Case       | All uppercase                      | `"hi".upper()` → `"HI"`                      |
| `lower()`                       | Case       | All lowercase                      | `"HI".lower()` → `"hi"`                      |
| `title()`                       | Case       | Title Case                         | `"hello world".title()` → `"Hello World"`    |
| `capitalize()`                  | Case       | First char upper                   | `"hi there".capitalize()` → `"Hi there"`     |
| `casefold()`                    | Case       | Aggressive lowercase (i18n)        | `"Straße".casefold()` → `"strasse"`          |
| `swapcase()`                    | Case       | Swap upper↔lower                   | `"Hi".swapcase()` → `"hI"`                   |
| `find(sub, start, end)`         | Searching  | Index of first sub (-1 if missing) | `"hello".find("ll")` → `2`                   |
| `rfind(sub)`                    | Searching  | Index of last sub                  | `"abab".rfind("ab")` → `2`                   |
| `index(sub)`                    | Searching  | Like find but raises ValueError    | `"hello".index("ll")` → `2`                  |
| `count(sub)`                    | Searching  | Count non-overlapping occurrences  | `"aaa".count("aa")` → `1`                    |
| `startswith(prefix)`            | Testing    | Starts with prefix?                | `"hello".startswith("he")` → `True`          |
| `endswith(suffix)`              | Testing    | Ends with suffix?                  | `"hello".endswith("lo")` → `True`            |
| `isdigit()`                     | Testing    | All digits?                        | `"123".isdigit()` → `True`                   |
| `isalpha()`                     | Testing    | All letters?                       | `"abc".isalpha()` → `True`                   |
| `isalnum()`                     | Testing    | All alphanumeric?                  | `"abc123".isalnum()` → `True`                |
| `isspace()`                     | Testing    | All whitespace?                    | `"  ".isspace()` → `True`                    |
| `isupper()` / `islower()`       | Testing    | All upper/lower?                   | `"HI".isupper()` → `True`                    |
| `center(width, fill)`           | Formatting | Center-pad                         | `"hi".center(10, "-")` → `"----hi----"`      |
| `ljust(width)` / `rjust(width)` | Formatting | Left/right pad                     | `"hi".ljust(5)` → `"hi   "`                  |
| `zfill(width)`                  | Formatting | Zero-pad left                      | `"42".zfill(5)` → `"00042"`                  |
| `encode(encoding)`              | Encoding   | To bytes                           | `"hi".encode("utf-8")` → `b'hi'`             |
| `format(*args, **kw)`           | Formatting | Format string                      | `"{} {}".format("a","b")` → `"a b"`          |
| `removeprefix(p)` (3.9+)        | Trimming   | Remove prefix                      | `"TestCase".removeprefix("Test")` → `"Case"` |
| `removesuffix(s)` (3.9+)        | Trimming   | Remove suffix                      | `"file.txt".removesuffix(".txt")` → `"file"` |

---

#### `collections.deque` — Double-Ended Queue (O(1) both ends)

| Method             | What it does                      | Returns     | Example                |
| ------------------ | --------------------------------- | ----------- | ---------------------- |
| `append(x)`        | Add to right end                  | `None`      | `d.append(3)`          |
| `appendleft(x)`    | Add to left end                   | `None`      | `d.appendleft(0)`      |
| `pop()`            | Remove from right end             | Element     | `d.pop()`              |
| `popleft()`        | Remove from left end              | Element     | `d.popleft()`          |
| `extend(iter)`     | Extend right end                  | `None`      | `d.extend([4,5])`      |
| `extendleft(iter)` | Extend left end (reversed)        | `None`      | `d.extendleft([0,-1])` |
| `rotate(n)`        | Rotate n steps right (neg = left) | `None`      | `d.rotate(1)`          |
| `count(x)`         | Count occurrences                 | `int`       | `d.count(2)`           |
| `remove(x)`        | Remove first occurrence           | `None`      | `d.remove(2)`          |
| `reverse()`        | Reverse in place                  | `None`      | `d.reverse()`          |
| `clear()`          | Remove all                        | `None`      | `d.clear()`            |
| `copy()`           | Shallow copy                      | New `deque` | `d.copy()`             |
| `maxlen`           | Max size (read-only property)     | `int\|None` | `deque(maxlen=100)`    |

> **When to use:** Use `deque` over `list` when you need fast appends/pops from **both ends** (e.g., BFS queue, sliding window, recent-N buffer). `list.pop(0)` is O(n); `deque.popleft()` is O(1).

---

#### `collections.defaultdict` — Dict with Auto-Default Values

| Method             | What it does                                        | Example                    |
| ------------------ | --------------------------------------------------- | -------------------------- |
| `default_factory`  | Attribute — callable that provides default          | `defaultdict(list)`        |
| `__missing__(key)` | Called when key not found → calls `default_factory` | Auto                       |
| All `dict` methods | Inherits everything from `dict`                     | `.get()`, `.items()`, etc. |

```python
from collections import defaultdict

# Group items
groups = defaultdict(list)
for item in [("a",1), ("b",2), ("a",3)]:
    groups[item[0]].append(item[1])
# {'a': [1, 3], 'b': [2]}

# Count items
counts = defaultdict(int)
for char in "mississippi":
    counts[char] += 1
# {'m': 1, 'i': 4, 's': 4, 'p': 2}

# Nested defaults
tree = defaultdict(lambda: defaultdict(list))
tree["users"]["admins"].append("Tushar")
```

---

#### `collections.Counter` — Counting Hashable Objects

| Method            | What it does                           | Returns       | Example                                               |
| ----------------- | -------------------------------------- | ------------- | ----------------------------------------------------- |
| `most_common(n)`  | Top n elements by count                | `list[tuple]` | `Counter("abbc").most_common(1)` → `[('b',2)]`        |
| `elements()`      | Iterator of elements repeated by count | `iterator`    | `list(Counter(a=2,b=1).elements())` → `['a','a','b']` |
| `subtract(other)` | Subtract counts (can go negative)      | `None`        | `c.subtract({"a": 1})`                                |
| `update(other)`   | Add counts                             | `None`        | `c.update("aab")`                                     |
| `total()` (3.10+) | Sum of all counts                      | `int`         | `Counter(a=2,b=3).total()` → `5`                      |
| `c + d`           | Add counters (drops zero/negative)     | New `Counter` |                                                       |
| `c - d`           | Subtract (drops zero/negative)         | New `Counter` |                                                       |
| `c & d`           | Min of corresponding counts            | New `Counter` |                                                       |
| `c \| d`          | Max of corresponding counts            | New `Counter` |                                                       |

```python
from collections import Counter

words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
c = Counter(words)
print(c.most_common(2))       # [('apple', 3), ('banana', 2)]
print(c["apple"])              # 3
print(c["missing"])            # 0  — no KeyError!
```

---

#### `collections.OrderedDict` — Dict Remembering Insertion Order

| Method                      | What it does                                   | Returns  |
| --------------------------- | ---------------------------------------------- | -------- |
| `move_to_end(k, last=True)` | Move key to end (or beginning if `last=False`) | `None`   |
| `popitem(last=True)`        | Pop from end (or beginning if `last=False`)    | `(k, v)` |
| All `dict` methods          | Same as regular dict                           | —        |

> **Note:** Since Python 3.7, regular `dict` preserves insertion order. Use `OrderedDict` only when you need `move_to_end()` or order-sensitive equality (`OrderedDict` considers order in `==`; regular `dict` doesn't).

---

#### `heapq` — Min-Heap Operations (via regular `list`)

| Function                   | What it does                      | Returns | Example                    |
| -------------------------- | --------------------------------- | ------- | -------------------------- |
| `heapify(lst)`             | Convert list to heap **in place** | `None`  | `heapq.heapify([3,1,2])`   |
| `heappush(heap, x)`        | Push item onto heap               | `None`  | `heapq.heappush(h, 5)`     |
| `heappop(heap)`            | Pop smallest item                 | Element | `heapq.heappop(h)`         |
| `heappushpop(heap, x)`     | Push then pop (faster than both)  | Element | `heapq.heappushpop(h, 3)`  |
| `heapreplace(heap, x)`     | Pop then push (faster than both)  | Element | `heapq.heapreplace(h, 3)`  |
| `nlargest(n, iter, key=)`  | N largest elements                | `list`  | `heapq.nlargest(3, nums)`  |
| `nsmallest(n, iter, key=)` | N smallest elements               | `list`  | `heapq.nsmallest(3, nums)` |

```python
import heapq

# Min-heap
h = [5, 3, 1, 4, 2]
heapq.heapify(h)            # [1, 2, 5, 4, 3]  — heap property, NOT sorted
heapq.heappush(h, 0)        # [0, 2, 1, 4, 3, 5]
print(heapq.heappop(h))     # 0 — always pops the smallest

# Max-heap trick — negate values
max_heap = []
heapq.heappush(max_heap, -5)
heapq.heappush(max_heap, -3)
print(-heapq.heappop(max_heap))   # 5 — largest

# Priority queue with tuples
tasks = []
heapq.heappush(tasks, (1, "critical"))
heapq.heappush(tasks, (3, "low"))
heapq.heappush(tasks, (2, "medium"))
print(heapq.heappop(tasks))      # (1, 'critical')
```

📌 **TLDR:** "Know `list.append` vs `extend`, `dict.setdefault` vs `get`, `set.discard` vs `remove`, `deque.appendleft`/`popleft` for O(1) both ends, `Counter.most_common()` for frequency counting, and `heapq` for priority queues. These come up constantly in both interviews and coding rounds."

---

### 9.25 `__slots__` — Memory Optimization & Interview Favorite

> **📣 Definition:** _"By default, every Python object stores its attributes in a per-instance `__dict__` dictionary — flexible but memory-hungry. `__slots__` replaces this dict with a fixed-size struct of slot descriptors, saving 30-50% memory per instance. The trade-off: you can't add arbitrary attributes at runtime. Use it when you have millions of instances of the same class (ORM rows, data points, game entities). It also makes attribute access slightly faster (direct offset vs dict lookup)."_

```python
# === Without __slots__ (default) ===
class PointDefault:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = PointDefault(1, 2)
p.__dict__                 # {'x': 1, 'y': 2}  ← per-instance dict (56+ bytes overhead)
p.z = 3                    # ✅ works — can add arbitrary attributes


# === With __slots__ ===
class PointSlots:
    __slots__ = ('x', 'y')

    def __init__(self, x, y):
        self.x = x
        self.y = y

p = PointSlots(1, 2)
p.__dict__                 # ❌ AttributeError — no __dict__ exists!
p.z = 3                    # ❌ AttributeError — can't add attributes not in __slots__


# === Memory comparison ===
import sys

default = PointDefault(1, 2)
slotted = PointSlots(1, 2)

sys.getsizeof(default) + sys.getsizeof(default.__dict__)  # ~152 bytes
sys.getsizeof(slotted)                                     # ~56 bytes  ← ~60% less!

# With 1 million instances:
# Without slots: ~152 MB
# With slots:    ~56 MB   ← saves ~96 MB
```

**When to use / not use:**

| Use `__slots__`                                       | Don't use `__slots__`                               |
| ----------------------------------------------------- | --------------------------------------------------- |
| Millions of instances (ORM-like objects, data points) | Classes where you dynamically add attributes        |
| Performance-critical inner loops                      | Classes using multiple inheritance (tricky)         |
| `dataclass(slots=True)` in Python 3.10+               | When you need `__dict__` for serialization/pickling |

**Gotchas:**

```python
# Gotcha 1: Inheritance — child must also define __slots__
class Point3D(PointSlots):
    __slots__ = ('z',)       # only NEW attributes! x, y inherited from parent

# Gotcha 2: If child doesn't define __slots__, it gets __dict__ back
class BadChild(PointSlots):
    pass                      # gets __dict__ — defeats the purpose

# Gotcha 3: __slots__ + dataclass (Python 3.10+)
from dataclasses import dataclass

@dataclass(slots=True)        # auto-generates __slots__ from fields
class Vector:
    x: float
    y: float
    z: float
```

📌 **TLDR:** "`__slots__` replaces per-instance `__dict__` with fixed slots — saves 30-50% memory, slightly faster attribute access, but can't add arbitrary attributes. Use when creating millions of instances. In 3.10+, use `@dataclass(slots=True)` for the best of both worlds."

---

### 9.26 How `dict` Works Internally — Hash Tables & Collision Resolution

> **📣 Definition:** _"Python's `dict` is a hash table. When you do `d[key] = value`, Python calls `hash(key)` to get an integer, computes `hash % table_size` to find a slot index, and stores the key-value pair there. Collisions (two keys mapping to the same slot) are resolved via **open addressing with probing** — Python looks for the next available slot using a perturbation-based probe sequence. Lookup is O(1) average, O(n) worst case (all keys collide). Dicts resize (double) when they're ~2/3 full to keep collisions rare."_

```python
# === Simplified mental model ===
# hash("name") → 1234567890
# index = 1234567890 % 8 → 2       (table starts at size 8)
# store ("name", "Tushar") at slot 2

# === What happens on a collision ===
# hash("name") % 8 → slot 2 (occupied by "name")
# hash("mane") % 8 → slot 2 (same slot! collision!)
# Python probes: try slot 2 → occupied → try slot f(2) → try slot f(f(2))...
# Probe formula (CPython): next_index = (5 * index + perturb + 1) % table_size
#                           perturb >>= 5  (shifts on each probe)
# This spreads probes across the table better than linear probing

# === Resize ===
# When the table is ~2/3 full, Python allocates a new table (2× size)
# and rehashes all existing entries into the new table.
# This is why dict insertion is amortized O(1), not strict O(1).

# === Compact dict (Python 3.6+) ===
# Before 3.6: hash table stored key+value+hash inline → sparse, wastes memory
# After 3.6:  separate dense array (insertion order) + sparse index table
# Result: 20-25% less memory AND preserves insertion order (guaranteed 3.7+)
```

**The hash-lookup algorithm (interview walkthrough):**

```
d = {"name": "Tushar", "age": 30}

d["name"]  →  Step-by-step:
  1. Compute hash:    h = hash("name")  →  e.g., 1234567890
  2. Compute index:   i = h % table_size  →  e.g., 2
  3. Check slot 2:    is it empty? → KeyError
                      does key match? → hash match AND key equality (==) → return value
                      hash matches but key differs? → COLLISION → probe next slot
  4. Probe sequence:  i = (5*i + perturb + 1) % size; perturb >>= 5
  5. Repeat step 3 at new index until found or empty slot
```

**Why `hash()` AND `==` are both needed:**

```python
# Two objects with the same hash are NOT necessarily equal (collision)
# Python checks: hash(a) == hash(b) AND a == b
# If hashes differ → definitely not equal (fast rejection)
# If hashes match → must check equality (slower, but rare with good hash function)

# This is why if you override __eq__, you MUST also override __hash__:
class User:
    def __init__(self, id, name):
        self.id = id
        self.name = name

    def __eq__(self, other):
        return self.id == other.id

    def __hash__(self):
        return hash(self.id)   # must be consistent with __eq__
```

📌 **TLDR:** "Python dict = hash table with open addressing. `hash(key) % size` finds the slot; collisions resolved by probing (not chaining). Resizes at 2/3 capacity. Lookup: check hash first (fast), then `==` (only if hash matches). Since 3.7, dicts preserve insertion order via a compact two-table layout. Always pair `__eq__` with `__hash__`."

---

### 9.27 Dict Keys — Why Only Hashable (Immutable) Types?

> **📣 Definition:** _"Dict keys must be **hashable** — meaning they implement `__hash__()` and the hash value never changes during the object's lifetime. All immutable built-in types (str, int, float, tuple of immutables, frozenset) are hashable. Mutable types (list, dict, set) are NOT hashable because if you mutated a key after insertion, its hash would change, and the dict could never find it again — it would look in the wrong slot forever."_

```python
# === Hashable (can be dict keys) ===
d = {}
d["hello"] = 1             # ✅ str — immutable, hashable
d[42] = 2                  # ✅ int — immutable, hashable
d[(1, 2, 3)] = 3           # ✅ tuple of immutables — hashable
d[frozenset({1, 2})] = 4   # ✅ frozenset — immutable set variant
d[True] = 5                # ✅ bool (subclass of int)

# === NOT hashable (cannot be dict keys) ===
d[[1, 2, 3]] = 6           # ❌ TypeError: unhashable type: 'list'
d[{1, 2}] = 7              # ❌ TypeError: unhashable type: 'set'
d[{"a": 1}] = 8            # ❌ TypeError: unhashable type: 'dict'

# === Why? The hash MUST be stable ===
# Imagine if lists were hashable:
key = [1, 2, 3]
# d[key] = "value"          # hash([1,2,3]) → slot 5
# key.append(4)             # key is now [1,2,3,4]
# d[key]                    # hash([1,2,3,4]) → slot 9 → KeyError! Data is lost at slot 5

# === Tuple gotcha: tuple of mutables is NOT hashable ===
hash((1, 2, 3))            # ✅ works
hash((1, [2, 3]))          # ❌ TypeError — tuple contains a list!

# === Custom objects as dict keys ===
# By default, custom objects ARE hashable (hash = id(), eq = identity)
# If you override __eq__, you MUST override __hash__ too:
class Point:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))

d = {Point(1, 2): "origin-adjacent"}   # ✅ works
```

📌 **TLDR:** "Dict keys must be hashable (stable `__hash__`). Immutable types (str, int, tuple, frozenset) are hashable. Mutable types (list, dict, set) aren't — mutating a key would break the hash table lookup. Custom objects are hashable by default (hash=id); override `__hash__` + `__eq__` together for value-based equality."

---

### 9.28 List Comprehension vs `for` Loop — Performance & Readability

> **📣 Definition:** _"List comprehensions are not just syntactic sugar — they're faster than equivalent `for` loops because the iteration and append happen in optimized C code inside CPython, avoiding the overhead of repeated `list.append()` method lookups and function calls. Typically 20-40% faster. But readability is the primary goal — use comprehensions for simple transforms, `for` loops for complex logic with side effects."_

```python
import timeit

# === Performance comparison ===
n = 1_000_000

# For loop
def with_loop():
    result = []
    for i in range(n):
        result.append(i * 2)
    return result

# List comprehension
def with_comp():
    return [i * 2 for i in range(n)]

# Generator expression (lazy, memory-efficient)
def with_gen():
    return sum(i * 2 for i in range(n))

# timeit results (typical):
# with_loop: ~85ms
# with_comp: ~55ms   ← ~35% faster
# with_gen:  ~65ms   ← lazy, no list allocation

# === WHY comprehensions are faster ===
# 1. No .append() method lookup on each iteration
# 2. List size is pre-allocated (CPython optimizes known-length comprehensions)
# 3. Bytecode: LIST_APPEND opcode vs LOAD_ATTR + CALL_FUNCTION for .append()
```

**When to use which:**

| Use                      | When                                                                                      |
| ------------------------ | ----------------------------------------------------------------------------------------- |
| **List comprehension**   | Simple transform/filter: `[x*2 for x in items if x > 0]`                                  |
| **Generator expression** | Large data, only need to iterate once: `sum(x*2 for x in items)`                          |
| **`for` loop**           | Side effects (print, API calls, mutations), complex multi-step logic                      |
| **`map()` + `filter()`** | Applying existing functions: `map(str.upper, names)` — slightly faster than comprehension |

```python
# ✅ Comprehension — clean, readable
squares = [x**2 for x in range(10)]
even_squares = [x**2 for x in range(10) if x % 2 == 0]
flat = [x for row in matrix for x in row]

# ✅ For loop — complex logic, side effects
results = []
for user in users:
    if user.is_active:
        data = fetch_profile(user.id)      # side effect: API call
        if data["score"] > threshold:
            results.append(transform(data))

# ❌ DON'T: comprehension with side effects (hard to read, bad practice)
[send_email(u) for u in users]   # creates useless list just for side effects
# ✅ DO: use a for loop
for u in users:
    send_email(u)

# ❌ DON'T: nested comprehension beyond 2 levels
result = [f(x, y, z) for x in xs for y in ys for z in zs if g(x, y, z)]
# ✅ DO: use a for loop for readability
```

📌 **TLDR:** "List comprehensions are 20-40% faster than `for` + `append()` due to optimized C-level bytecode. Use comprehensions for simple transforms/filters. Use `for` loops for side effects and complex logic. Use generator expressions when you don't need the full list in memory. Never use comprehensions just for side effects."

---

### 9.29 Synchronization Primitives in Python

> **📣 Definition:** _"When multiple threads access shared state, you need synchronization primitives to prevent race conditions. Python's `threading` module provides: Lock (mutual exclusion), RLock (re-entrant lock), Semaphore (limited concurrent access), Event (thread signaling), Condition (wait-for-notification), and Barrier (sync point for N threads). `asyncio` has async equivalents for coroutines."_

```python
import threading

# === 1. Lock — basic mutual exclusion ===
lock = threading.Lock()
counter = 0

def increment():
    global counter
    with lock:                    # acquires on enter, releases on exit (even on exception)
        counter += 1              # only one thread at a time

# Without lock: counter += 1 is actually READ → ADD → WRITE (3 ops, race condition!)


# === 2. RLock — re-entrant lock (same thread can acquire multiple times) ===
rlock = threading.RLock()

def outer():
    with rlock:
        inner()                   # same thread re-acquires — OK with RLock, DEADLOCK with Lock!

def inner():
    with rlock:
        print("inner")


# === 3. Semaphore — allow N concurrent threads ===
sem = threading.Semaphore(5)      # max 5 threads at once

def limited_access(url):
    with sem:                     # blocks if 5 threads already inside
        return requests.get(url)  # at most 5 concurrent HTTP requests


# === 4. Event — thread signaling (one-shot flag) ===
ready = threading.Event()

def worker():
    ready.wait()                  # blocks until event is set
    print("Go!")

def controller():
    # ... setup ...
    ready.set()                   # wake up ALL waiting threads


# === 5. Condition — wait + notify (producer-consumer) ===
condition = threading.Condition()
queue = []

def producer():
    with condition:
        queue.append(item)
        condition.notify()        # wake ONE waiting consumer

def consumer():
    with condition:
        while not queue:
            condition.wait()      # releases lock, sleeps, re-acquires on wake
        return queue.pop(0)


# === 6. Barrier — sync point for N threads ===
barrier = threading.Barrier(3)    # wait until 3 threads reach this point

def phase_worker():
    do_phase_1()
    barrier.wait()                # all 3 threads must reach here before any proceeds
    do_phase_2()                  # all start phase 2 together
```

**Quick comparison table:**

| Primitive       | Purpose                     | Use Case                                    |
| --------------- | --------------------------- | ------------------------------------------- |
| **`Lock`**      | Mutual exclusion (1 thread) | Protecting shared counters, data structures |
| **`RLock`**     | Re-entrant lock             | Recursive functions that need locking       |
| **`Semaphore`** | Allow N concurrent          | Connection pools, rate limiting             |
| **`Event`**     | One-shot signal             | "Start" signal, ready notification          |
| **`Condition`** | Wait + notify               | Producer-consumer queues                    |
| **`Barrier`**   | Sync N threads              | Phased computation, parallel testing        |

**Asyncio equivalents:**

```python
import asyncio
# asyncio.Lock(), asyncio.Semaphore(), asyncio.Event(), asyncio.Condition()
# Same API, but use `async with` and `await`:
lock = asyncio.Lock()
async with lock:
    await do_something()
```

📌 **TLDR:** "6 sync primitives: `Lock` (1 thread), `RLock` (re-entrant), `Semaphore(N)` (N concurrent), `Event` (flag signal), `Condition` (wait/notify for producer-consumer), `Barrier` (sync point). Always use `with lock:` (context manager) to guarantee release. Asyncio has async equivalents."

---

### 9.30 `asyncio.Future` vs `asyncio.Task` — What's the Difference?

> **📣 Definition:** _"A `Future` is a placeholder for a result that doesn't exist yet — it represents an eventual value. A `Task` is a subclass of `Future` that wraps a coroutine and schedules it on the event loop. You almost never create `Future` directly — it's a low-level primitive. You create `Task` objects via `asyncio.create_task()`. The key difference: a Task actively runs a coroutine; a Future passively waits for someone to set its result."_

```python
import asyncio

# === Task (what you use 99% of the time) ===
async def fetch_data():
    await asyncio.sleep(1)
    return {"id": 1, "name": "Tushar"}

async def main():
    # create_task() schedules the coroutine to run on the event loop
    task = asyncio.create_task(fetch_data())   # starts running immediately!
    print(type(task))          # <class 'asyncio.Task'>
    print(isinstance(task, asyncio.Future))  # True — Task IS a Future

    result = await task        # wait for completion, get result
    print(result)              # {"id": 1, "name": "Tushar"}


# === Future (low-level, rarely used directly) ===
async def manual_future_example():
    loop = asyncio.get_event_loop()
    future = loop.create_future()      # empty placeholder

    # Someone must manually set the result:
    async def resolver():
        await asyncio.sleep(1)
        future.set_result(42)          # manually resolve

    asyncio.create_task(resolver())
    result = await future              # waits until set_result is called
    print(result)                      # 42


# === When you see Futures: bridging callback-based APIs ===
# Libraries like aiohttp internally use Futures to wrap callback-style I/O
# You as a user work with Tasks (via create_task or gather)
```

**Comparison:**

| Aspect            | `Future`                               | `Task`                                |
| ----------------- | -------------------------------------- | ------------------------------------- |
| **What**          | Placeholder for a future result        | Future + scheduled coroutine          |
| **Creates**       | `loop.create_future()`                 | `asyncio.create_task(coro())`         |
| **Runs code?**    | ❌ No — just a container               | ✅ Yes — actively drives a coroutine  |
| **Result set by** | Manual: `future.set_result(x)`         | Automatic: when coroutine returns     |
| **When to use**   | Bridging callback APIs, low-level libs | Everything else — your default choice |
| **Inherits**      | Base class                             | Subclass of `Future`                  |

📌 **TLDR:** "Task = Future + a running coroutine. Use `asyncio.create_task()` to schedule coroutines — it returns a Task that runs concurrently. Future is a low-level placeholder you rarely create directly. Task IS a Future (subclass). Use `gather()` to run multiple tasks concurrently."

---

### 9.31 Sharing Data Between Processes (IPC in Python)

> **📣 Definition:** _"Since each process has its own memory space, you can't just share variables like with threads. Python provides several Inter-Process Communication (IPC) mechanisms: `Queue` (thread/process-safe FIFO), `Pipe` (two-way channel between two processes), `Value`/`Array` (shared memory for simple types), and `Manager` (proxy objects for dicts, lists, etc. across processes). Choose based on complexity: Queue for most cases, shared memory for performance-critical numeric data."_

```python
from multiprocessing import Process, Queue, Pipe, Value, Array, Manager

# === 1. Queue — safest, most common ===
def producer(q):
    q.put({"id": 1, "data": "hello"})
    q.put("DONE")

def consumer(q):
    while True:
        item = q.get()            # blocks until item available
        if item == "DONE":
            break
        print(f"Got: {item}")

q = Queue()
p1 = Process(target=producer, args=(q,))
p2 = Process(target=consumer, args=(q,))
p1.start(); p2.start()
p1.join(); p2.join()


# === 2. Pipe — fast, two endpoints (duplex by default) ===
def sender(conn):
    conn.send([1, 2, 3])
    conn.close()

parent_conn, child_conn = Pipe()
p = Process(target=sender, args=(child_conn,))
p.start()
print(parent_conn.recv())    # [1, 2, 3]
p.join()


# === 3. Value / Array — shared memory (fast, for numeric types) ===
counter = Value('i', 0)       # 'i' = int, shared between processes
arr = Array('d', [1.0, 2.0, 3.0])   # 'd' = double

def increment(shared_counter, n):
    for _ in range(n):
        with shared_counter.get_lock():    # must lock!
            shared_counter.value += 1


# === 4. Manager — high-level shared objects (dict, list, etc.) ===
def worker(shared_dict, key, value):
    shared_dict[key] = value

with Manager() as manager:
    d = manager.dict()         # proxy dict shared across processes
    l = manager.list()         # proxy list shared across processes

    procs = [Process(target=worker, args=(d, f"key_{i}", i)) for i in range(5)]
    for p in procs: p.start()
    for p in procs: p.join()

    print(dict(d))             # {'key_0': 0, 'key_1': 1, ...}
```

**IPC comparison:**

| Method              | Speed   | Complexity | Use Case                                              |
| ------------------- | ------- | ---------- | ----------------------------------------------------- |
| **`Queue`**         | Medium  | Low        | General producer-consumer, task distribution          |
| **`Pipe`**          | Fast    | Low        | Two-process communication only                        |
| **`Value`/`Array`** | Fastest | Medium     | Shared counters, numeric arrays (need manual locking) |
| **`Manager`**       | Slowest | Low        | Shared dicts/lists (proxy overhead, but convenient)   |

📌 **TLDR:** "Processes have separate memory — use IPC to share data. `Queue` for most cases (safe, easy). `Pipe` for fast two-process channels. `Value`/`Array` for shared numeric data (fastest, needs locking). `Manager` for shared dicts/lists (convenient but slowest). Always prefer Queue unless you have a specific performance need."

---

### 1 Real-World Parallelization — Reading Files & Sending to API

> **📣 Definition:** _"The classic interview question: 'You have 10,000 files to read and send to an API. How do you parallelize this?' The answer depends on the bottleneck. File reading is I/O-bound (threads or async work). API calls are I/O-bound (async is best). CPU-heavy processing needs multiprocessing. The production pattern: use a thread/process pool for file reading + async HTTP client for API calls, with backpressure to avoid overwhelming the target."_

```python
# === Pattern 1: ThreadPoolExecutor (simplest, good for moderate scale) ===
import requests
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path

def process_file(filepath: Path) -> dict:
    """Read file → send to API → return result."""
    data = filepath.read_text()
    response = requests.post("https://api.example.com/ingest", json={"content": data})
    return {"file": filepath.name, "status": response.status_code}

files = list(Path("./data").glob("*.json"))

with ThreadPoolExecutor(max_workers=20) as pool:
    futures = {pool.submit(process_file, f): f for f in files}
    for future in as_completed(futures):
        result = future.result()
        print(f"{result['file']}: {result['status']}")


# === Pattern 2: asyncio + aiohttp (best for high-volume API calls) ===
import asyncio
import aiohttp
import aiofiles

async def process_file_async(session: aiohttp.ClientSession, filepath: Path):
    async with aiofiles.open(filepath, 'r') as f:
        data = await f.read()
    async with session.post("https://api.example.com/ingest", json={"content": data}) as resp:
        return {"file": filepath.name, "status": resp.status}

async def main():
    sem = asyncio.Semaphore(50)       # limit concurrent requests (backpressure!)

    async def limited(session, filepath):
        async with sem:
            return await process_file_async(session, filepath)

    async with aiohttp.ClientSession() as session:
        tasks = [limited(session, f) for f in files]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        for r in results:
            if isinstance(r, Exception):
                print(f"Error: {r}")
            else:
                print(f"{r['file']}: {r['status']}")

asyncio.run(main())


# === Pattern 3: multiprocessing + threads (CPU processing + API calls) ===
from multiprocessing import Pool
from concurrent.futures import ThreadPoolExecutor

def cpu_heavy_process(filepath: Path) -> dict:
    """Read file → heavy processing → return processed data."""
    data = filepath.read_text()
    processed = expensive_transform(data)  # CPU-bound (parsing, ML, etc.)
    return {"file": filepath.name, "data": processed}

def send_batch(items: list):
    """Send processed items to API (I/O-bound — use threads)."""
    with ThreadPoolExecutor(max_workers=10) as pool:
        return list(pool.map(
            lambda item: requests.post(API_URL, json=item).status_code,
            items,
        ))

# Step 1: Process files with multiprocessing (CPU-bound)
with Pool(processes=4) as pool:
    processed = pool.map(cpu_heavy_process, files)

# Step 2: Send to API with threading (I/O-bound)
# Batch to avoid overwhelming the API
batch_size = 100
for i in range(0, len(processed), batch_size):
    batch = processed[i:i + batch_size]
    send_batch(batch)
```

**Decision framework:**

| Scenario                            | Best Tool                                | Why                                               |
| ----------------------------------- | ---------------------------------------- | ------------------------------------------------- |
| Read files + send to API (I/O only) | `asyncio` + `aiohttp`                    | Handles 1000s of concurrent I/O ops on one thread |
| Simple file processing + API        | `ThreadPoolExecutor`                     | Easy, good enough for 100s of files               |
| CPU-heavy processing + API calls    | `multiprocessing` + `ThreadPoolExecutor` | CPU work in processes, I/O in threads             |
| Must not overwhelm the API          | `asyncio.Semaphore(N)`                   | Limits concurrent requests (backpressure)         |

📌 **TLDR:** "For I/O-heavy parallelization (read files + call APIs): use `asyncio` + `aiohttp` with a `Semaphore` for backpressure — handles thousands of concurrent operations. For CPU + I/O: multiprocessing for CPU work, threads for API calls. Always add backpressure (semaphore or batch size) to avoid overwhelming the target API."

---

### 10. Python Type Hints & `pydantic`

> **📣 Definition:** _"Pydantic is a Python library that uses type hints for runtime data validation, serialization, and settings management. Define a class inheriting from `BaseModel` with typed fields, and Pydantic validates incoming data, coerces types where safe, raises clear errors on mismatches, and serializes to JSON. Pydantic v2 is written in Rust — 5-50x faster than v1. It's the foundation FastAPI uses for request validation and response serialization, and it doubles as a fantastic settings manager via `pydantic-settings`."_

```python
from pydantic import BaseModel, Field, field_validator

class User(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=0, le=150)
    email: str

    @field_validator("email")
    @classmethod
    def validate_email(cls, v):
        if "@" not in v:
            raise ValueError("Invalid email")
        return v.lower()
```

**📌 TLDR**: "Type hints + Pydantic = runtime data validation + serialization + documentation. FastAPI is built on Pydantic. Use it for API request/response models, settings management, and data parsing. Pydantic v2 is written in Rust and is dramatically faster."

---
Most Likely Questions

These are the exact phrasings interviewers use. Practice answering each in 60-90 seconds.

### Python Core

**Q: "What's the difference between `__init__` and `__new__`?"**
A: `__new__` creates the object (runs first, returns the instance). `__init__` initializes it (runs after, returns None). You override `__new__` for singletons or immutable types like tuples; `__init__` for normal class setup.

**Q: "Explain how decorators work under the hood."**
A: A decorator is a function that takes a function and returns a new function. `@my_decorator` is sugar for `func = my_decorator(func)`. The wrapper closes over the original function and adds behavior before/after calling it. Always use `@functools.wraps` to preserve metadata.

**Q: "What happens when you do `a = [1, 2, 3]; b = a; b.append(4)`?"**
A: `a` and `b` reference the same list object. `b.append(4)` mutates the shared list, so `a` is also `[1, 2, 3, 4]`. Python passes object references, not values. To get an independent copy: `b = a.copy()` or `b = a[:]`.

**Q: "Walk me through what happens when Python imports a module."**
A: Python checks `sys.modules` cache first. If not cached, it locates the file, compiles to bytecode (cached as `.pyc`), creates a module object, executes the top-level code, and adds it to `sys.modules`. Subsequent imports return the cached object — that's why module-level code runs only once.

**Q: "Mutable default arguments — what's the trap?"**
A: `def f(items=[])` — the default list is created **once** at function definition, not per call. So `f()` then `f()` shares the same list. Fix: `def f(items=None): items = items or []`.

### Concurrency

**Q: "I have a script that downloads 100 URLs. How would you speed it up?"**
A: I/O-bound, so I'd use `asyncio` with `httpx.AsyncClient` and `asyncio.gather()` — should drop from sequential ~100×latency to roughly the slowest single request. If async isn't an option, `ThreadPoolExecutor` with ~20 workers gives similar speedup. Multiprocessing would be overkill — too much overhead for I/O.

**Q: "When does threading actually help in Python given the GIL?"**
A: When threads spend time waiting on something outside Python's bytecode — network I/O, file I/O, database queries, or C extensions that release the GIL (NumPy, Pillow). The GIL is released during these waits, so other threads can run. For pure Python CPU work, threading gives no parallelism.

**Q: "What's a race condition? Give an example."**
A: When two threads access shared state without coordination, and the result depends on timing. Classic example: `counter += 1` is actually three operations (read, add, write). Two threads can both read the same value, both increment, both write — losing one increment. Fix: `threading.Lock()` or use atomic operations.

### Async

**Q: "I see `async def` everywhere in this codebase but the API is slow. What might be wrong?"**
A: Most likely a sync I/O call hiding inside an async function — `requests.get()` instead of `httpx.AsyncClient`, or `time.sleep()` instead of `asyncio.sleep()`, or a sync DB driver. These block the entire event loop, serializing all requests on that worker. I'd grep for sync libraries and swap them, or wrap blocking calls in `loop.run_in_executor()`.

**Q: "Explain the difference between `await` and `asyncio.gather()`."**
A: `await` runs one coroutine and waits for it — it's sequential within a function. `asyncio.gather()` schedules multiple coroutines concurrently on the event loop and waits for all to complete. If you have two independent I/O calls, sequential awaits take `t1 + t2`; gather takes `max(t1, t2)`.

**Q: "What's the difference between concurrency and parallelism in Python specifically?"**
A: Concurrency in Python means juggling many tasks on one thread via asyncio — great for I/O. Parallelism means running on multiple cores via multiprocessing — needed for CPU-bound work because the GIL serializes Python bytecode in threads. Production FastAPI typically combines both: 4 Uvicorn workers (parallelism) × 1 event loop each (concurrency).

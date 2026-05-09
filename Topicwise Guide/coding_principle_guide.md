# Coding principles

### 📑 Table of Contents

1. [1. SOLID Principles](#1-solid-principles)
2. [2. DRY, KISS & YAGNI](#2-dry-kiss--yagni--the-complementary-principles)
3. [3. SAGA Pattern & Other Design Patterns](#3-saga-pattern--other-design-patterns)
4. [4. Idempotency & REST API Design](#4-idempotency--rest-api-design)

## 1. SOLID Principles

> **📣 Interview-ready definition:** _"SOLID is five object-oriented design principles that help write maintainable code. **S**ingle Responsibility — a class should do one thing. **O**pen-Closed — open for extension, closed for modification. **L**iskov Substitution — subclasses should be drop-in replacements for their parents. **I**nterface Segregation — keep interfaces small and focused. **D**ependency Inversion — depend on abstractions, not concrete implementations. The goal isn't to memorize them but to recognize when code violates them and refactor toward better design."_

### 🧑‍🍳 Layman Explanation

SOLID is like **5 rules for building good LEGO sets**:

- **S** — Each LEGO piece does one thing (a wheel is a wheel, not a wheel+window combo).
- **O** — You can add new LEGO pieces without breaking existing ones.
- **L** — Any red 2x4 brick should work wherever a 2x4 brick is expected.
- **I** — Don't force a spaceship builder to also know boat instructions.
- **D** — The instruction manual says "attach a wheel" — not "attach _this specific_ wheel from _this specific_ box."

### 🔧 Technical Explanation

---

#### S — Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change."

**Real-world e-commerce example:**

```python
# 🚫 BAD: Order class handles everything
class Order:
    def __init__(self, items):
        self.items = items

    def calculate_total(self):                  # business logic
        return sum(i.price for i in self.items)

    def save_to_database(self):                 # persistence
        db.execute("INSERT INTO orders ...")

    def send_confirmation_email(self):          # notification
        smtp.send(self.customer_email, ...)

    def generate_invoice_pdf(self):             # report generation
        pdf = PDFGenerator()
        pdf.create(...)

# Problems:
# - Change in email service → modify Order class
# - Change in DB schema → modify Order class
# - Change in PDF library → modify Order class
# - Hard to test in isolation
# - Hard to reuse parts (e.g., emails for refunds)

# ✅ GOOD: Each class has ONE reason to change
class Order:
    def __init__(self, items):
        self.items = items

    def calculate_total(self):
        return sum(i.price for i in self.items)

class OrderRepository:
    def save(self, order: Order):
        db.execute("INSERT INTO orders ...")

class OrderNotifier:
    def send_confirmation(self, order: Order, email: str):
        smtp.send(email, ...)

class InvoiceGenerator:
    def generate_pdf(self, order: Order):
        ...

# Now: testing OrderNotifier doesn't require a real Order or database.
```

**Common SRP violations to call out:**

- A "God" class with 30+ methods
- A class with mixed concerns (data + I/O + UI logic)
- A model class that also sends emails (very common Django anti-pattern!)

---

#### O — Open/Closed Principle (OCP)

> "Software entities should be **open for extension** but **closed for modification**."

**Real-world example: Discount calculator**

```python
# 🚫 BAD: Adding a new discount type means modifying existing code
class DiscountCalculator:
    def calculate(self, order, discount_type):
        if discount_type == "percentage":
            return order.total * 0.1
        elif discount_type == "fixed":
            return 100
        elif discount_type == "bogo":           # Adding new type? Edit this class!
            return order.total / 2
        # Each new type = code change + retest existing types

# ✅ GOOD: Extension via polymorphism (Strategy Pattern!)
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, order) -> float:
        pass

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percent): self.percent = percent
    def calculate(self, order):
        return order.total * (self.percent / 100)

class FixedDiscount(DiscountStrategy):
    def __init__(self, amount): self.amount = amount
    def calculate(self, order):
        return self.amount

class BogoDiscount(DiscountStrategy):           # NEW class, no existing code touched!
    def calculate(self, order):
        return order.total / 2

# Usage
discount = PercentageDiscount(10)
final_price = order.total - discount.calculate(order)
```

**OCP enabler patterns:** Strategy, Template Method, Observer, Decorator. Most often achieved via **abstraction + dependency injection**.

---

#### L — Liskov Substitution Principle (LSP)

> "Objects of a superclass should be **replaceable** with objects of a subclass without breaking the application."

If you have `def f(x: Animal)`, passing any subclass of Animal must work — without surprises.

**Classic Rectangle/Square violation (interview gold):**

```python
# 🚫 BAD: Square IS-A Rectangle mathematically, but not behaviorally
class Rectangle:
    def __init__(self, w, h):
        self.width = w
        self.height = h

    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

    def area(self): return self.width * self.height

class Square(Rectangle):
    def set_width(self, w):
        self.width = w
        self.height = w        # forces height == width!

    def set_height(self, h):
        self.width = h
        self.height = h

# This breaks reasonable expectations:
def stretch_rectangle(r: Rectangle):
    r.set_width(10)
    r.set_height(5)
    assert r.area() == 50      # ✅ passes for Rectangle

stretch_rectangle(Square(2, 2))   # ❌ FAILS! area = 25, not 50
# Caller assumed independent width/height — Square broke that assumption.
```

**Real-world example: Bird flying violation**

```python
# 🚫 BAD: Penguin IS-A Bird, but can't fly
class Bird:
    def fly(self): print("flying")

class Sparrow(Bird): pass

class Penguin(Bird):
    def fly(self):
        raise NotImplementedError("Penguins can't fly!")  # ❌ LSP violation

def make_birds_fly(birds: list[Bird]):
    for bird in birds:
        bird.fly()    # crashes if any Penguin is in the list

# ✅ GOOD: Refactor hierarchy to match actual behavior
class Bird: pass

class FlyingBird(Bird):
    def fly(self): print("flying")

class Sparrow(FlyingBird): pass
class Penguin(Bird): pass     # No fly() method, no false promise
```

**LSP smell test:** "Can I pass any subclass to code expecting the parent without surprises?" If your subclass throws NotImplementedError, weakens contracts, or strengthens preconditions — you're violating LSP.

---

#### I — Interface Segregation Principle (ISP)

> "Clients should not be forced to depend on methods they do not use."

**Real-world example: Multi-function devices**

```python
# 🚫 BAD: One fat interface
class Machine(ABC):
    @abstractmethod
    def print_doc(self, doc): pass
    @abstractmethod
    def scan_doc(self, doc): pass
    @abstractmethod
    def fax_doc(self, doc): pass

# A simple printer is forced to implement scan and fax!
class OldPrinter(Machine):
    def print_doc(self, doc): print("printing")
    def scan_doc(self, doc): raise NotImplementedError("can't scan")
    def fax_doc(self, doc): raise NotImplementedError("can't fax")

# ✅ GOOD: Small, focused interfaces
class Printer(ABC):
    @abstractmethod
    def print_doc(self, doc): pass

class Scanner(ABC):
    @abstractmethod
    def scan_doc(self, doc): pass

class Faxer(ABC):
    @abstractmethod
    def fax_doc(self, doc): pass

class OldPrinter(Printer):                     # only what it needs
    def print_doc(self, doc): print("printing")

class ModernAllInOne(Printer, Scanner, Faxer): # composes multiple interfaces
    def print_doc(self, doc): ...
    def scan_doc(self, doc): ...
    def fax_doc(self, doc): ...
```

**ISP smell test:** Are subclasses raising `NotImplementedError`? Are they implementing methods as no-ops? That's an interface that's too fat.

---

#### D — Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. **Both should depend on abstractions.** Abstractions should not depend on details. Details should depend on abstractions."

**Real-world example: Notification service**

```python
# 🚫 BAD: High-level OrderService directly depends on a concrete email class
class SMTPEmailSender:
    def send(self, to, subject, body):
        smtplib.SMTP("smtp.gmail.com").sendmail(...)

class OrderService:
    def __init__(self):
        self.email = SMTPEmailSender()         # tightly coupled to SMTP!

    def confirm_order(self, order):
        self.email.send(order.email, "Order confirmed", "...")

# Problems:
# - Want to switch to SendGrid? Modify OrderService.
# - Want to test? Must mock the entire SMTP library.
# - Want to add SMS? Add another concrete dependency.

# ✅ GOOD: Depend on an abstraction
class Notifier(ABC):
    @abstractmethod
    def send(self, to: str, subject: str, body: str): pass

class SMTPEmailSender(Notifier):
    def send(self, to, subject, body): ...

class SendGridSender(Notifier):
    def send(self, to, subject, body): ...

class SMSNotifier(Notifier):
    def send(self, to, subject, body): ...

class OrderService:
    def __init__(self, notifier: Notifier):    # depends on abstraction
        self.notifier = notifier

    def confirm_order(self, order):
        self.notifier.send(order.email, "Order confirmed", "...")

# Now we can swap implementations freely
order_service = OrderService(notifier=SendGridSender())   # production
test_service = OrderService(notifier=MockNotifier())      # testing
```

**DIP enablers:** Dependency Injection (constructor, setter, or via DI containers like FastAPI's `Depends`), abstract base classes, protocols (Python 3.8+ structural typing).

**Where you've seen DIP applied:**

- FastAPI's `Depends()` system
- Django's `AUTH_USER_MODEL` setting (depend on a user model abstraction)
- pytest fixtures
- Repository pattern in DDD

---

#### SOLID Cheat Card

| Principle | Smell to look for                                                 | Refactor toward                              |
| --------- | ----------------------------------------------------------------- | -------------------------------------------- |
| **SRP**   | A class with mixed concerns (DB + email + business logic)         | Split by responsibility                      |
| **OCP**   | If/elif chain checking types                                      | Polymorphism (Strategy pattern)              |
| **LSP**   | Subclass throws `NotImplementedError` or breaks parent's contract | Fix the hierarchy or use composition         |
| **ISP**   | Subclass forced to implement methods it doesn't need              | Split interface into smaller ones            |
| **DIP**   | High-level code instantiating low-level concretes                 | Inject via constructor + abstract base class |

### 📌 TLDR for Interviews

> "SOLID = 5 design principles for maintainable code. **S**ingle Responsibility: one reason to change — split DB/email/business logic into separate classes. **O**pen-Closed: extend without modifying — use Strategy pattern instead of if/elif type chains. **L**iskov: subtypes must honor parent contracts — Square shouldn't extend Rectangle if it changes width/height behavior. **I**nterface Segregation: small focused interfaces — don't force a Printer to implement scan/fax. **D**ependency Inversion: depend on abstractions, not concretes — inject a `Notifier` interface, not a hardcoded SMTP client. FastAPI's `Depends()` and Django's `AUTH_USER_MODEL` are real examples of D in practice."

---

### 2. DRY, KISS & YAGNI — The Complementary Principles

> **📣 Definition:** _"DRY (Don't Repeat Yourself), KISS (Keep It Simple, Stupid), and YAGNI (You Aren't Gonna Need It) complement SOLID. DRY says every piece of knowledge should have a single, unambiguous representation — duplicate logic leads to inconsistent behavior when only one copy gets updated. KISS says prefer the simplest solution that works — clever code is expensive to maintain. YAGNI says don't build features until you actually need them — speculative abstraction wastes time and adds complexity. Together with SOLID, these form the core design philosophy for maintainable software."_

**DRY — Don't Repeat Yourself:**

```python
# ❌ BAD: Same validation logic duplicated
def create_user(email, name):
    if not email or "@" not in email:
        raise ValueError("Invalid email")
    # ... create user ...

def update_user(email, name):
    if not email or "@" not in email:      # duplicated!
        raise ValueError("Invalid email")
    # ... update user ...

# ✅ GOOD: Single source of truth
def validate_email(email: str) -> str:
    if not email or "@" not in email:
        raise ValueError("Invalid email")
    return email.lower().strip()

def create_user(email, name):
    email = validate_email(email)
    # ... create user ...

def update_user(email, name):
    email = validate_email(email)          # one place to fix
    # ... update user ...
```

**DRY gotcha — don't over-DRY:**

```python
# ❌ OVER-DRY: Forcing unrelated things together because they "look similar"
# Just because two functions have similar structure doesn't mean
# they represent the same KNOWLEDGE. DRY is about duplicated KNOWLEDGE,
# not duplicated code characters.

# Rule of Three: tolerate duplication until you see 3 instances.
# Then abstract — because now the pattern is real, not coincidental.
```

**KISS — Keep It Simple:**

```python
# ❌ BAD: Overly clever one-liner
result = (lambda f, n: f(f, n))(lambda s, n: n if n <= 1 else n * s(s, n-1), 5)

# ✅ GOOD: Simple, readable
def factorial(n: int) -> int:
    if n <= 1:
        return 1
    return n * factorial(n - 1)
```

**YAGNI — You Aren't Gonna Need It:**

```python
# ❌ BAD: Building a plugin system "in case we need it later"
class PaymentProcessor:
    def __init__(self, strategy_factory, audit_logger, event_bus, retry_policy, ...):
        # 80% of this won't be used for months

# ✅ GOOD: Build what you need NOW, refactor when requirements emerge
class PaymentProcessor:
    def charge(self, amount: Decimal, card_token: str) -> str:
        return stripe.charge(amount, card_token)
```

**Comparison table:**

| Principle | Core Idea                       | Violation Smell                                 | Fix                             |
| --------- | ------------------------------- | ----------------------------------------------- | ------------------------------- |
| **DRY**   | Single source of truth          | Copy-pasted logic, constants in multiple places | Extract function/class/constant |
| **KISS**  | Simplest solution that works    | Clever one-liners, premature abstraction        | Rewrite for readability         |
| **YAGNI** | Don't build what you don't need | Unused abstractions, speculative features       | Delete it; add when needed      |
| **SOLID** | Maintainable OO design          | God classes, fat interfaces, tight coupling     | See SOLID breakdown above       |

📌 **TLDR:** "DRY = don't duplicate knowledge (but don't over-DRY coincidental similarity). KISS = prefer simplicity over cleverness. YAGNI = don't build features until needed. Together with SOLID, these are the 4 pillars of clean software design. The goal of all of them: **code that's easy to change**."

---

## 3. SAGA Pattern & Other Design Patterns

> **📣 Interview-ready definition:** _"Design patterns are reusable solutions to recurring software design problems. The classic Gang of Four patterns split into Creational (how objects are created — Singleton, Factory, Builder), Structural (how they're composed — Adapter, Decorator, Facade, Proxy), and Behavioral (how they interact — Strategy, Observer, Command, State). For distributed systems, key patterns are SAGA (cross-service transactions with compensating actions), CQRS (separating reads from writes), and Outbox (reliable event publishing). The skill isn't using patterns — it's recognizing when a problem matches a pattern that's been solved well before."_

### 🧑‍🍳 Layman Explanation

**Design patterns** are reusable solutions to common problems — like cooking techniques (sauté, braise, deglaze) that chefs reuse across recipes. You don't reinvent them every time; you recognize the situation and apply the right technique.

**SAGA** = Booking a vacation with flights + hotel + car rental. If the car rental fails after you've already booked the flight and hotel, you don't just give up — you cancel the hotel, cancel the flight (compensating actions), and roll everything back gracefully. Each step has an "undo button."

### 🔧 Technical Explanation

Patterns split into 3 categories:

1. **Creational** — How objects are created (Factory, Singleton, Builder, Prototype)
2. **Structural** — How objects are composed (Adapter, Decorator, Facade, Proxy, Composite)
3. **Behavioral** — How objects interact (Strategy, Observer, Command, Iterator, State, Template, Chain of Responsibility)

Plus modern **distributed systems patterns**: SAGA, CQRS, Event Sourcing, Outbox, Circuit Breaker, etc.

---

#### 📋 Interview Quick Reference — All Patterns in One Place

> **How to answer pattern questions in interviews:** _"Name it → one sentence what it does → one real-world example → one-line code hint. Don't write full implementations unless asked."_

**CREATIONAL (how objects are created):**

| Pattern              | What it does                                                           | Example to say                                                       | Code hint                                                                       |
| -------------------- | ---------------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| **Singleton**        | Ensures only ONE instance of a class exists globally                   | "Logger, DB connection pool — one shared instance across the app"    | `__new__` checks `cls._instance is None`                                        |
| **Factory Method**   | A method decides which class to instantiate based on input             | "NotificationFactory.create('email') returns EmailNotifier"          | `mapping[type]()` — dict of type→class                                          |
| **Abstract Factory** | Creates _families_ of related objects without knowing concrete classes | "UIFactory creates matching Button + Input for Web or Mobile"        | Factory ABC with `create_button()`, `create_input()` — each platform implements |
| **Builder**          | Constructs complex objects step-by-step with a fluent API              | "HttpRequestBuilder().url(...).method('POST').header(...).build()"   | Chained `.method()` calls returning `self`, final `.build()`                    |
| **Prototype**        | Clones an existing object instead of creating from scratch             | "Document template — clone it for each report instead of rebuilding" | `copy.deepcopy(prototype)`                                                      |

**STRUCTURAL (how objects are composed):**

| Pattern       | What it does                                                     | Example to say                                                    | Code hint                                                                 |
| ------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **Adapter**   | Wraps an incompatible interface to match the one you expect      | "Legacy payment API uses cents — Adapter converts to rupees"      | Wrapper class translating `pay(rupees)` → `make_payment(cents)`           |
| **Decorator** | Adds behavior to objects dynamically without subclassing         | "Coffee + Milk + Sugar — each wraps the previous, adding cost"    | `Milk(Sugar(Coffee())).cost()` — each calls `self.wrapped.cost() + extra` |
| **Facade**    | Provides a simple interface to a complex subsystem               | "OrderFacade.place_order() hides inventory + payment + shipping"  | One class with one method orchestrating multiple services                 |
| **Proxy**     | Stand-in for another object — controls access, adds lazy loading | "ImageProxy loads the real image only on first `.display()` call" | Holds `_real_object = None`, creates it on first use                      |
| **Flyweight** | Shares common data across many objects to save memory            | "Font('Arial', 12) — returns cached instance, not a new one"      | `__new__` with a class-level `_cache` dict                                |

**BEHAVIORAL (how objects interact):**

| Pattern                     | What it does                                                      | Example to say                                                                       | Code hint                                                                      |
| --------------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| **Strategy**                | Swap algorithms at runtime via interchangeable classes            | "DiscountStrategy — switch between PercentageDiscount and FixedDiscount"             | Constructor takes `strategy: ABC`, calls `strategy.calculate()`                |
| **Observer**                | Notifies multiple listeners when state changes (pub/sub)          | "Django signals, JS events — 'order placed' triggers email + inventory + log"        | `subject.attach(observer)` → `subject.notify()` loops observers                |
| **Command**                 | Encapsulates a request as an object — enables undo/queue          | "Text editor — each keystroke is a Command with `execute()` + `undo()`"              | Command ABC with `execute()` / `undo()`, stored in a history list              |
| **State**                   | Object behavior changes based on internal state (state machine)   | "Order: Pending→Paid→Shipped→Delivered — each state defines `next()` and `cancel()`" | `self.state = PaidState()` — state objects handle transitions                  |
| **Template Method**         | Defines algorithm skeleton; subclasses fill in specific steps     | "DataPipeline.run() calls extract→transform→load — subclasses implement each"        | Base class with `def run(): self.extract(); self.transform(); self.load()`     |
| **Chain of Responsibility** | Passes a request through a chain of handlers until one handles it | "Django middleware — Auth → RateLimit → BusinessLogic, each can short-circuit"       | `handler.set_next(next_handler)` — each calls `super().handle()` to pass along |
| **Iterator**                | Provides sequential access without exposing internals             | "Python's `for x in obj` — built into the language via `__iter__`/`__next__`"        | `__iter__` returns self, `__next__` returns next or raises `StopIteration`     |

**DISTRIBUTED SYSTEMS:**

| Pattern             | What it does                                               | Example to say                                                                         |
| ------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| **SAGA**            | Manages distributed transactions with compensating actions | "Order fails at shipping → undo payment → undo inventory (each step has an undo)"      |
| **CQRS**            | Separates read and write models for independent scaling    | "Write to Postgres, read from Elasticsearch — optimized for each"                      |
| **Event Sourcing**  | Stores state as append-only events, not current state      | "Bank ledger — every transaction is an event, balance = replay all events"             |
| **Outbox**          | Reliable event publishing alongside DB writes              | "Write order + event to same DB in one transaction, separate worker publishes"         |
| **Circuit Breaker** | Stops calling a failing service, fails fast instead        | "After 5 failures, circuit opens for 30s — returns fallback, doesn't pile up timeouts" |

---

#### A. Creational Patterns — Deep Dive

**1. Singleton** — Only one instance ever exists

```python
# Use case: DB connection pool, app config, logger
class DatabasePool:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._init_pool()
        return cls._instance

    def _init_pool(self):
        self.connections = [create_conn() for _ in range(10)]

# Usage
pool1 = DatabasePool()
pool2 = DatabasePool()
assert pool1 is pool2   # same instance!
```

**When to use:** Shared resources (config, logger, DB pool, cache client). **Caution:** Singletons are global state — make testing hard. Often a code smell. Modern Python: use a module-level variable or DI instead.

**2. Factory Method** — A method decides which class to instantiate

```python
# Use case: Object creation logic varies by input
class Notification(ABC):
    @abstractmethod
    def send(self, msg): pass

class EmailNotification(Notification):
    def send(self, msg): print(f"Email: {msg}")

class SMSNotification(Notification):
    def send(self, msg): print(f"SMS: {msg}")

class PushNotification(Notification):
    def send(self, msg): print(f"Push: {msg}")

class NotificationFactory:
    @staticmethod
    def create(channel: str) -> Notification:
        mapping = {
            "email": EmailNotification,
            "sms": SMSNotification,
            "push": PushNotification,
        }
        cls = mapping.get(channel)
        if not cls:
            raise ValueError(f"Unknown channel: {channel}")
        return cls()

# Usage
notifier = NotificationFactory.create("email")
notifier.send("Hello!")
```

**When to use:** When object creation depends on runtime input (config, user choice). Avoids if/elif chains scattered across the codebase.

**3. Builder** — Construct complex objects step by step

```python
# Use case: Object with many optional parameters
class HttpRequest:
    def __init__(self):
        self.url = None
        self.method = "GET"
        self.headers = {}
        self.body = None
        self.timeout = 30

class HttpRequestBuilder:
    def __init__(self):
        self.request = HttpRequest()

    def url(self, url):
        self.request.url = url
        return self        # return self for chaining

    def method(self, method):
        self.request.method = method
        return self

    def header(self, key, value):
        self.request.headers[key] = value
        return self

    def body(self, body):
        self.request.body = body
        return self

    def timeout(self, seconds):
        self.request.timeout = seconds
        return self

    def build(self):
        return self.request

# Usage — fluent, readable
req = (HttpRequestBuilder()
       .url("https://api.example.com")
       .method("POST")
       .header("Authorization", "Bearer xyz")
       .body({"key": "value"})
       .timeout(60)
       .build())
```

**When to use:** Objects with 5+ optional parameters where you want immutability or step-wise construction. Pythonic alternative: `dataclasses` with default values + `replace()`.

**4. Abstract Factory** — Creates families of related objects without specifying concrete classes

```python
# Use case: UI toolkit that must work across platforms (Web, Mobile, Desktop)
class Button(ABC):
    @abstractmethod
    def render(self): pass

class TextInput(ABC):
    @abstractmethod
    def render(self): pass

# Concrete products — Web family
class WebButton(Button):
    def render(self): return "<button>Click</button>"

class WebTextInput(TextInput):
    def render(self): return '<input type="text"/>'

# Concrete products — Mobile family
class MobileButton(Button):
    def render(self): return "[Mobile Button]"

class MobileTextInput(TextInput):
    def render(self): return "[Mobile Input]"

# Abstract Factory
class UIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button: pass
    @abstractmethod
    def create_text_input(self) -> TextInput: pass

class WebUIFactory(UIFactory):
    def create_button(self): return WebButton()
    def create_text_input(self): return WebTextInput()

class MobileUIFactory(UIFactory):
    def create_button(self): return MobileButton()
    def create_text_input(self): return MobileTextInput()

# Client code works with ANY factory — doesn't know concrete classes
def build_form(factory: UIFactory):
    btn = factory.create_button()
    inp = factory.create_text_input()
    return f"{inp.render()} {btn.render()}"

build_form(WebUIFactory())     # '<input type="text"/> <button>Click</button>'
build_form(MobileUIFactory())  # '[Mobile Input] [Mobile Button]'
```

**Abstract Factory vs Factory Method:**

- **Factory Method**: one method, one product → `NotificationFactory.create("email")`
- **Abstract Factory**: a family of methods, a family of related products → `UIFactory.create_button()` + `UIFactory.create_text_input()`

**When to use:** When you need to create families of related objects that must be used together (UI themes, database adapters, OS-specific components).

**5. Prototype** — Clone existing objects instead of creating from scratch

```python
import copy

# Use case: Expensive-to-create objects (DB-loaded templates, game entities)
class DocumentTemplate:
    def __init__(self, title, sections, formatting, metadata):
        self.title = title
        self.sections = sections            # complex nested structure
        self.formatting = formatting        # loaded from config
        self.metadata = metadata            # expensive to compute

    def clone(self):
        """Deep copy — fully independent clone."""
        return copy.deepcopy(self)

# Create one expensive "prototype"
base_template = DocumentTemplate(
    title="Report",
    sections=["Intro", "Analysis", "Conclusion"],
    formatting={"font": "Arial", "size": 12, "margins": {"top": 1, "bottom": 1}},
    metadata={"version": 3, "schema": load_schema()},  # expensive!
)

# Clone is cheap — no DB calls, no schema loading
monthly_report = base_template.clone()
monthly_report.title = "Monthly Report - May 2026"

quarterly_report = base_template.clone()
quarterly_report.title = "Q2 2026 Report"
quarterly_report.sections.append("Quarterly Summary")

# base_template is unaffected by either clone's changes
```

**When to use:** When object creation is expensive (DB lookups, file parsing, network calls) and you need many similar instances. Python's `copy.deepcopy()` IS the Prototype pattern. Also: game entity spawning, document templates, config presets.

---

#### B. Structural Patterns

**1. Adapter** — Make incompatible interfaces work together

```python
# Use case: Wrapping a third-party library's interface to match yours
class LegacyPaymentSystem:                     # third-party, can't modify
    def make_payment(self, amt_in_cents):
        print(f"Charging {amt_in_cents} cents")

class PaymentGateway(ABC):                     # YOUR interface
    @abstractmethod
    def pay(self, amount_rupees: float): pass

class LegacyPaymentAdapter(PaymentGateway):    # the adapter
    def __init__(self, legacy: LegacyPaymentSystem):
        self.legacy = legacy

    def pay(self, amount_rupees: float):
        self.legacy.make_payment(int(amount_rupees * 100))   # rupees → paise

# Usage
gateway = LegacyPaymentAdapter(LegacyPaymentSystem())
gateway.pay(500.0)                             # works with YOUR interface
```

**When to use:** Integrating with legacy code or third-party APIs whose interface doesn't match yours.

**2. Decorator (Structural, different from Python's `@decorator`)** — Add behavior to objects dynamically

```python
# Use case: Coffee shop add-ons
class Coffee(ABC):
    @abstractmethod
    def cost(self) -> float: pass
    @abstractmethod
    def description(self) -> str: pass

class SimpleCoffee(Coffee):
    def cost(self): return 50
    def description(self): return "Coffee"

class CoffeeDecorator(Coffee):                 # base decorator
    def __init__(self, coffee: Coffee):
        self.coffee = coffee

class Milk(CoffeeDecorator):
    def cost(self): return self.coffee.cost() + 10
    def description(self): return self.coffee.description() + " + Milk"

class Sugar(CoffeeDecorator):
    def cost(self): return self.coffee.cost() + 5
    def description(self): return self.coffee.description() + " + Sugar"

# Usage — stack decorators dynamically!
order = Sugar(Milk(SimpleCoffee()))
print(order.description(), "₹", order.cost())  # "Coffee + Milk + Sugar ₹65"
```

**When to use:** Adding optional behaviors at runtime (logging, caching, validation) without subclass explosion.

**3. Facade** — Simplified interface to a complex subsystem

```python
# Use case: Hide complexity of multiple services behind one API
class OrderFacade:
    def __init__(self):
        self.inventory = InventoryService()
        self.payment = PaymentService()
        self.shipping = ShippingService()
        self.notifier = NotificationService()

    def place_order(self, user, items):
        if not self.inventory.check_availability(items):
            raise OutOfStockError()
        self.inventory.reserve(items)
        payment_id = self.payment.charge(user, sum(i.price for i in items))
        tracking = self.shipping.create_shipment(user.address, items)
        self.notifier.send_confirmation(user.email, tracking)
        return {"payment_id": payment_id, "tracking": tracking}

# Client doesn't need to know about 4 different services
result = OrderFacade().place_order(user, items)
```

**When to use:** When clients shouldn't know subsystem complexity. Common in API gateways, service layers.

**4. Proxy** — Stand-in for another object (lazy loading, access control, caching)

```python
# Use case: Lazy-loaded image
class Image(ABC):
    @abstractmethod
    def display(self): pass

class RealImage(Image):
    def __init__(self, filename):
        self.filename = filename
        self._load_from_disk()                 # expensive!

    def _load_from_disk(self):
        print(f"Loading {self.filename}")

    def display(self):
        print(f"Displaying {self.filename}")

class ImageProxy(Image):
    def __init__(self, filename):
        self.filename = filename
        self._real_image = None                # lazy

    def display(self):
        if self._real_image is None:
            self._real_image = RealImage(self.filename)  # load on first use
        self._real_image.display()

# Usage
img = ImageProxy("huge_photo.jpg")             # cheap — no load yet
img.display()                                  # loads + displays
img.display()                                  # cached — just displays
```

**When to use:** Lazy initialization, access control (security proxy), caching, logging method calls. Django's `lazy()` and ORM's `QuerySet` use proxy-like behavior.

**5. Flyweight** — Share data to support large numbers of fine-grained objects efficiently

```python
# Use case: Text editor (millions of characters sharing font/style objects)
# Instead of each character storing its own Font(name, size, color), share them.

class Font:
    """Flyweight — shared, immutable intrinsic state."""
    _cache = {}                             # flyweight pool

    def __new__(cls, name: str, size: int, bold: bool = False):
        key = (name, size, bold)
        if key not in cls._cache:
            instance = super().__new__(cls)
            instance.name = name
            instance.size = size
            instance.bold = bold
            cls._cache[key] = instance
        return cls._cache[key]

# Usage
f1 = Font("Arial", 12)
f2 = Font("Arial", 12)
f3 = Font("Arial", 14)          # different size = different instance

print(f1 is f2)                  # True — same object, shared!
print(f1 is f3)                  # False — different intrinsic state
print(len(Font._cache))          # 2 — only 2 Font objects for potentially millions of characters

# Each character stores:
# - extrinsic state: position, character value (unique per character)
# - intrinsic state: font (shared via flyweight)
class Character:
    def __init__(self, char: str, font: Font, x: int, y: int):
        self.char = char
        self.font = font         # shared flyweight
        self.x = x               # unique per character
        self.y = y

# 1 million characters, but only ~10 Font objects in memory
```

**Flyweight vs Singleton:**

- **Singleton**: exactly ONE instance of a class
- **Flyweight**: MANY shared instances, one per unique state combination

**Real-world Flyweight in Python:**

- **String interning**: Python caches small strings — `"hello" is "hello"` → True
- **Small integer cache**: Python caches integers -5 to 256
- **`sys.intern()`**: Explicitly intern strings for dict key optimization
- **Connection pools**: Shared pool of DB connections (limited instances, reused)

**When to use:** When you have millions of objects with significant shared state — text rendering (fonts), game maps (tile types), icon sets, cached immutable configs.

---

#### C. Behavioral Patterns

**1. Strategy** — Interchangeable algorithms (the OCP enabler)

```python
# Already shown in SOLID/OCP — this IS the Strategy pattern
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data: list) -> list: pass

class QuickSort(SortStrategy):
    def sort(self, data): return sorted(data)  # simplified

class MergeSort(SortStrategy):
    def sort(self, data): ...

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self.strategy = strategy

    def execute(self, data):
        return self.strategy.sort(data)

# Switch algorithms at runtime
sorter = Sorter(QuickSort())
sorter.strategy = MergeSort()                  # change strategy mid-flight
```

**When to use:** Multiple ways to do the same thing (sort algorithms, payment methods, compression schemes, pricing rules).

**2. Observer (Pub/Sub)** — Notify multiple objects of state changes

```python
# Use case: Event-driven systems, UI updates, model lifecycle hooks
class Subject:
    def __init__(self):
        self._observers = []
        self._state = None

    def attach(self, observer):
        self._observers.append(observer)

    def notify(self):
        for observer in self._observers:
            observer.update(self._state)

    def set_state(self, state):
        self._state = state
        self.notify()

class EmailObserver:
    def update(self, state):
        print(f"Email: state changed to {state}")

class LogObserver:
    def update(self, state):
        print(f"Log: state = {state}")

# Usage
subject = Subject()
subject.attach(EmailObserver())
subject.attach(LogObserver())
subject.set_state("active")    # notifies both observers
```

**When to use:** Anywhere you need decoupled "X happened, others should react" — Django signals, JS events, Kafka pub/sub, Redux store subscribers. All applications of Observer.

**3. Command** — Encapsulate a request as an object

```python
# Use case: Undo/redo, task queues, audit logs
class Command(ABC):
    @abstractmethod
    def execute(self): pass
    @abstractmethod
    def undo(self): pass

class TextEditor:
    def __init__(self):
        self.text = ""

class WriteCommand(Command):
    def __init__(self, editor: TextEditor, text: str):
        self.editor = editor
        self.text = text

    def execute(self):
        self.editor.text += self.text

    def undo(self):
        self.editor.text = self.editor.text[:-len(self.text)]

# Usage with history
editor = TextEditor()
history = []

cmd = WriteCommand(editor, "Hello ")
cmd.execute()
history.append(cmd)

cmd2 = WriteCommand(editor, "World")
cmd2.execute()
history.append(cmd2)

# Undo last
history.pop().undo()                           # editor.text = "Hello "
```

**When to use:** Undo/redo, queueable operations (Celery tasks ARE Commands), macro recording, request logging.

**4. Iterator** — Sequential access without exposing internals
Already covered in detail in [Section 6](python_core_guide.md#6-iterators--generators). Python's iterator protocol (`__iter__` / `__next__`) is this pattern built into the language.

**5. State** — Object behavior changes based on internal state

```python
# Use case: Order workflow (Pending → Paid → Shipped → Delivered)
class OrderState(ABC):
    @abstractmethod
    def next(self, order): pass
    @abstractmethod
    def cancel(self, order): pass

class PendingState(OrderState):
    def next(self, order):
        order.state = PaidState()
        print("Pending → Paid")
    def cancel(self, order):
        order.state = CancelledState()

class PaidState(OrderState):
    def next(self, order):
        order.state = ShippedState()
        print("Paid → Shipped")
    def cancel(self, order):
        print("Refund needed!")
        order.state = CancelledState()

class ShippedState(OrderState):
    def next(self, order):
        order.state = DeliveredState()
        print("Shipped → Delivered")
    def cancel(self, order):
        raise Exception("Can't cancel shipped order!")

class DeliveredState(OrderState):
    def next(self, order): pass
    def cancel(self, order): raise Exception("Already delivered!")

class CancelledState(OrderState):
    def next(self, order): pass
    def cancel(self, order): pass

class Order:
    def __init__(self):
        self.state = PendingState()

    def progress(self): self.state.next(self)
    def cancel(self): self.state.cancel(self)

# Usage
o = Order()
o.progress()    # Pending → Paid
o.progress()    # Paid → Shipped
o.cancel()      # ERROR: Can't cancel shipped order!
```

**When to use:** Workflows, state machines, traffic lights, game character states. Replaces messy if/elif chains based on state.

**6. Template Method** — Define algorithm skeleton, let subclasses fill in steps

```python
# Use case: Algorithm with variable steps
class DataPipeline(ABC):
    def run(self):                             # template method (don't override)
        data = self.extract()
        cleaned = self.transform(data)
        self.load(cleaned)

    @abstractmethod
    def extract(self): pass
    @abstractmethod
    def transform(self, data): pass
    @abstractmethod
    def load(self, data): pass

class CSVPipeline(DataPipeline):
    def extract(self): return open("data.csv").read()
    def transform(self, data): return data.upper()
    def load(self, data): open("output.csv", "w").write(data)

class APIPipeline(DataPipeline):
    def extract(self): return requests.get("...").json()
    def transform(self, data): return [d for d in data if d["active"]]
    def load(self, data): db.bulk_insert(data)

# Both pipelines: extract → transform → load (skeleton fixed)
CSVPipeline().run()
APIPipeline().run()
```

**When to use:** Common algorithm structure with variable steps. Django's CBVs (e.g., `CreateView.post()`) use this heavily.

**7. Chain of Responsibility** — Pass request through a chain until handled

```python
# Use case: Middleware, validation chains, support tickets escalation
class Handler(ABC):
    def __init__(self):
        self.next: Handler = None

    def set_next(self, handler):
        self.next = handler
        return handler                         # for chaining

    def handle(self, request):
        if self.next:
            return self.next.handle(request)
        return None

class AuthHandler(Handler):
    def handle(self, request):
        if not request.get("token"):
            return "401 Unauthorized"
        return super().handle(request)

class RateLimitHandler(Handler):
    def handle(self, request):
        if request.get("count", 0) > 100:
            return "429 Too Many Requests"
        return super().handle(request)

class BusinessLogicHandler(Handler):
    def handle(self, request):
        return "200 OK: " + str(request)

# Build the chain
auth = AuthHandler()
auth.set_next(RateLimitHandler()).set_next(BusinessLogicHandler())

print(auth.handle({"token": "abc", "count": 5}))     # 200 OK
print(auth.handle({"count": 5}))                     # 401 Unauthorized
```

**When to use:** Django middleware, Express.js middleware, validation pipelines, event handlers.

---

#### D. Distributed Systems Patterns

**1. SAGA Pattern** — Manages distributed transactions across microservices without a global ACID transaction.

Two types:

1. **Choreography**: Each service publishes events; the next service reacts. No central coordinator. Simple but hard to debug.
2. **Orchestration**: A central **Saga Orchestrator** tells each service what to do. Easier to manage, single point of visibility.

```
Orchestration SAGA — E-commerce Order:
1. OrderService → CreateOrder          (compensate: CancelOrder)
2. PaymentService → ChargePayment      (compensate: RefundPayment)
3. InventoryService → ReserveStock     (compensate: ReleaseStock)
4. ShippingService → CreateShipment    (compensate: CancelShipment)

If Step 3 fails → execute compensations: RefundPayment, CancelOrder
```

**2. Other Critical Distributed Patterns:**

| Pattern                         | What it solves                     | One-liner                                                                                                                                       |
| ------------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| **CQRS**                        | Read/write optimization            | Separate models for commands (writes) and queries (reads). Scale them independently.                                                            |
| **Event Sourcing**              | Auditability, replay               | Store state as a sequence of events (append-only log), not current state. Rebuild state by replaying events.                                    |
| **Outbox Pattern**              | Reliable event publishing          | Write business data + event to same DB in one transaction. Separate process reads outbox table and publishes to message broker.                 |
| **Circuit Breaker**             | Cascading failures                 | If a downstream service fails repeatedly, "open" the circuit and fail fast instead of overwhelming it. Three states: Closed → Open → Half-Open. |
| **Strangler Fig**               | Monolith → Microservices migration | Gradually replace parts of a monolith by routing traffic to new services.                                                                       |
| **API Gateway**                 | Client simplification              | Single entry point for all microservices. Handles routing, auth, rate-limiting, aggregation.                                                    |
| **Sidecar**                     | Cross-cutting concerns             | Deploy a helper container alongside your service for logging, monitoring, service mesh (Istio/Envoy).                                           |
| **Retry + Exponential Backoff** | Transient failures                 | Retry failed requests with increasing delays: 1s, 2s, 4s, 8s... + jitter.                                                                       |
| **Bulkhead**                    | Resource isolation                 | Separate thread/connection pools per downstream so one slow service doesn't drain everything.                                                   |

---

#### E. Decision Tree — When to Use Which Pattern

| Problem                                                        | Pattern                                  |
| -------------------------------------------------------------- | ---------------------------------------- |
| "I have an if/elif chain switching on a type/algorithm"        | **Strategy** or **Factory**              |
| "I want to add behavior to objects dynamically"                | **Decorator** (structural)               |
| "I need to notify multiple things when state changes"          | **Observer** / Pub-Sub                   |
| "I need to execute, queue, log, and undo operations"           | **Command**                              |
| "Object behavior depends on a complex internal state"          | **State** machine                        |
| "I have a complex algorithm with steps that should vary"       | **Template Method**                      |
| "Multiple validators/handlers might process a request"         | **Chain of Responsibility** / Middleware |
| "I'm wrapping a legacy/third-party API with a clean interface" | **Adapter**                              |
| "I want lazy loading or access control on an expensive object" | **Proxy**                                |
| "I need to hide complexity of multiple subsystems"             | **Facade**                               |
| "Building an object with many optional parameters"             | **Builder** (or `dataclass` in Python)   |
| "I need exactly one instance shared across the app"            | **Singleton** (cautiously — prefer DI)   |
| "Object creation logic depends on runtime input"               | **Factory Method** / Abstract Factory    |
| "Distributed transaction across microservices"                 | **SAGA** + **Outbox**                    |
| "Optimize reads and writes separately at scale"                | **CQRS**                                 |
| "Auditable history, ability to replay state"                   | **Event Sourcing**                       |
| "Protect from cascading failures of downstream services"       | **Circuit Breaker** + **Bulkhead**       |
| "Migrate monolith to microservices gradually"                  | **Strangler Fig**                        |

### 📌 TLDR for Interviews

> "Design patterns are reusable solutions to recurring problems. **Creational** (Singleton, Factory, Builder) — how to create objects. **Structural** (Adapter, Decorator, Facade, Proxy) — how to compose them. **Behavioral** (Strategy, Observer, Command, State, Chain of Responsibility) — how they interact. For distributed systems: **SAGA** for cross-service transactions with compensating actions, **CQRS** for read/write separation, **Outbox** for reliable event publishing, **Circuit Breaker** for cascading failures. The goal isn't to use patterns — it's to recognize when you're solving a problem someone solved well before, and reach for the right one. Strategy + Factory together solve 80% of OO design problems; everything else is icing."

---

## 4. Idempotency & REST API Design

> **📣 Interview-ready definition:** _"Idempotency means doing the same operation multiple times produces the same result — like pressing an elevator button repeatedly. In HTTP, GET, PUT, and DELETE are idempotent by spec; POST is not, but you can make it idempotent using an `Idempotency-Key` header. Good REST API design uses nouns for resources (`/orders/123`), HTTP verbs for actions (GET/POST/PUT/PATCH/DELETE), correct status codes (200/201/400/404/422/429/500), proper versioning, pagination, and consistent error formats."_

### 🧑‍🍳 Layman Explanation

**Idempotency** = Pressing the elevator button 10 times has the same effect as pressing it once. No matter how many times you repeat the action, the result is the same. This is crucial when networks are unreliable and requests might be retried.

### 🔧 Technical Explanation

**HTTP Method Idempotency:**

| Method | Idempotent? | Safe?  | Use case                |
| ------ | ----------- | ------ | ----------------------- |
| GET    | ✅ Yes      | ✅ Yes | Retrieve data           |
| PUT    | ✅ Yes      | ❌ No  | Replace entire resource |
| DELETE | ✅ Yes      | ❌ No  | Remove resource         |
| PATCH  | ❌ No\*     | ❌ No  | Partial update          |
| POST   | ❌ No       | ❌ No  | Create resource         |

\*PATCH can be made idempotent with careful design.

**Implementing idempotency for POST:**

```python
# Client sends an Idempotency-Key header
@app.post("/payments")
async def create_payment(payment: PaymentRequest, idempotency_key: str = Header()):
    # Check if we've already processed this key
    existing = await redis.get(f"idem:{idempotency_key}")
    if existing:
        return json.loads(existing)  # return cached response

    result = await process_payment(payment)

    # Cache the result with a TTL (e.g., 24 hours)
    await redis.setex(f"idem:{idempotency_key}", 86400, json.dumps(result))
    return result
```

**REST API Design Best Practices:**

- **Resources as nouns**: `/users`, `/orders/{id}/items` — NOT `/getUser`, `/createOrder`.
- **HTTP verbs for actions**: GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE.
- **Versioning**: `/api/v1/users` or `Accept: application/vnd.api+json;version=1`.
- **Pagination**: `?page=2&limit=20` or cursor-based `?cursor=abc123&limit=20`.
- **Filtering/Sorting**: `?status=active&sort=-created_at`.
- **Error responses**: Consistent structure with HTTP status codes.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Email is required",
    "details": [{ "field": "email", "issue": "missing" }]
  }
}
```

- **HATEOAS** (Hypermedia): Include links to related actions in responses.
- **Rate Limiting**: Return `429 Too Many Requests` with `Retry-After` header.
- **Status codes**: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity, 429 Rate Limited, 500 Internal Error.

---
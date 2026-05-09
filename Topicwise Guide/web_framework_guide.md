# Web Framework Guide

## 📑 Table of Contents

**Django (Tech Lead Level)**

1. [1. Django — Complete Guide for Technical Lead Role](#1-django--complete-guide-for-technical-lead-role)
2. [2. Django Architecture — MVT (Model-View-Template)](#2-django-architecture--mvt-model-view-template)
3. [3. Django Request-Response Lifecycle (Critical for Tech Lead)](#3-django-request-response-lifecycle-critical-for-tech-lead)
4. [4. Models, Fields, and Relationships](#4-models-fields-and-relationships)
5. [5. Model Inheritance — Three Strategies](#5-model-inheritance--three-strategies)
6. [6. QuerySets — Lazy Evaluation & Chaining](#6-querysets--lazy-evaluation--chaining)
7. [7. `Q` Objects — Complex Query Logic (OR/AND/NOT)](#7-q-objects--complex-query-logic-orandnot)
8. [8. `F` Expressions — Atomic Updates & Field-to-Field Comparison](#8-f-expressions--atomic-updates--field-to-field-comparison)
9. [9. `select_related` vs `prefetch_related` — The N+1 Killer](#9-select_related-vs-prefetch_related--the-n1-killer)
10. [10. Aggregation, Annotation & Subqueries](#10-aggregation-annotation--subqueries)
11. [11. Custom Managers & QuerySets](#11-custom-managers--querysets)
12. [12. Signals — Power & Pitfalls](#12-signals--power--pitfalls)
13. [13. Middleware — Architecture & Custom Implementation](#13-middleware--architecture--custom-implementation)
14. [14. Class-Based Views (CBVs) vs Function-Based Views (FBVs)](#14-class-based-views-cbvs-vs-function-based-views-fbvs)
15. [15. Django REST Framework (DRF) — Production Patterns](#15-django-rest-framework-drf--production-patterns)
16. [16. DRF Serializers — Deep Dive](#16-drf-serializers--deep-dive)
17. [17. Transactions, `atomic`, and `on_commit`](#17-transactions-atomic-and-on_commit)
18. [18. Django Caching](#18-django-caching)
19. [19. Django Async — Async Views, ORM, and Concurrency](#19-django-async--async-views-orm-and-concurrency)
20. [20. Django Channels — WebSockets & Real-Time](#20-django-channels--websockets--real-time)
21. [21. Celery — Background Tasks & Scheduled Jobs](#21-celery--background-tasks--scheduled-jobs)
22. [22. Migrations — Production-Safe Strategies](#22-migrations--production-safe-strategies)
23. [23. Custom User Model & Authentication](#23-custom-user-model--authentication)
24. [24. Multi-Database Support & Database Routers](#24-multi-database-support--database-routers)
25. [25. Testing in Django](#25-testing-in-django)
26. [26. Django Security — What Tech Leads Must Know](#26-django-security--what-tech-leads-must-know)
27. [27. Performance & Scaling — Tech-Lead Talking Points](#27-performance--scaling--tech-lead-talking-points)
28. [28. Common Tech-Lead Interview Questions & How to Answer Them](#28-common-tech-lead-interview-questions--how-to-answer-them)
29. [29. API Versioning in Django / DRF](#29-api-versioning-in-django--drf)
30. [30. JWT Authentication & RBAC in Django](#30-jwt-authentication--rbac-in-django)
31. [31. Django Security — Extended Deep Dive](#31-django-security--extended-deep-dive)
32. [32. Migration Management — Versioning & Production Workflows](#32-migration-management--versioning--production-workflows)
33. [33. Horizontally Scaling a Django Application](#33-horizontally-scaling-a-django-application)

**FastAPI (Tech Lead Level)**

34. [34. FastAPI — Complete Guide for Technical Lead Role](#34-fastapi--complete-guide-for-technical-lead-role)
35. [35. ASGI vs WSGI — The Foundation](#35-asgi-vs-wsgi--the-foundation)
36. [36. Path Operations, Parameters & Routing](#36-path-operations-parameters--routing)
37. [37. Pydantic Models — Validation, Serialization, Settings](#37-pydantic-models--validation-serialization-settings)
38. [38. Dependency Injection — The Killer Feature](#38-dependency-injection--the-killer-feature)
39. [39. Async vs Sync Endpoints (Pitfall Heavy)](#39-async-vs-sync-endpoints-pitfall-heavy)
40. [40. Middleware — Custom and Built-in](#40-middleware--custom-and-built-in)
41. [41. Background Tasks vs Celery — When to Use What](#41-background-tasks-vs-celery--when-to-use-what)
42. [42. Authentication & Authorization (OAuth2 + JWT)](#42-authentication--authorization-oauth2--jwt)
43. [43. Database Integration — SQLAlchemy 2.0 + Alembic](#43-database-integration--sqlalchemy-20--alembic)
44. [44. WebSockets & Server-Sent Events](#44-websockets--server-sent-events)
45. [45. Testing — TestClient, AsyncClient, Fixtures](#45-testing--testclient-asyncclient-fixtures)
46. [46. Project Structure for Large Apps](#46-project-structure-for-large-apps)
47. [47. Production Deployment](#47-production-deployment)
48. [48. Performance Optimization & Pitfalls](#48-performance-optimization--pitfalls)
49. [49. FastAPI vs Django vs Flask — When to Choose What](#49-fastapi-vs-django-vs-flask--when-to-choose-what)
50. [50. Tech-Lead Interview Q&A](#50-tech-lead-interview-qa)
51. [51. 📌 The Master TLDR for FastAPI](#51--the-master-tldr-for-fastapi)
52. [52. Migration Management — Alembic Deep Dive & Versioning](#52-migration-management--alembic-deep-dive--versioning)
53. [53. Pydantic v2 — Deep Dive for Production](#53-pydantic-v2--deep-dive-for-production)
54. [54. Migration Alternatives & Advanced Strategies](#54-migration-alternatives--advanced-strategies)

## 1. Django — Complete Guide for Technical Lead Role

> **📣 Interview-ready definition:** _"Django is a 'batteries-included' Python web framework built around an MVT (Model-View-Template) architecture. It ships with an ORM, admin UI, authentication, sessions, forms, migrations, and security features out of the box, making it ideal for full-stack web apps and content-heavy systems. For tech-lead interviews, the focus shifts from syntax to production concerns: ORM optimization (`select_related`/`prefetch_related` to kill N+1 queries), signals and their pitfalls, custom user models, async views and ORM, transaction handling with `atomic()` and `on_commit()`, middleware design, scaling with Celery for background tasks, and migration safety in production."_

This section is exhaustive and ordered from foundational to advanced. Even if you've used Django for years, the deeper topics (custom managers, signals pitfalls, async ORM, Channels, multi-DB routing, query optimization) are what tech-lead interviews drill into.

---

### 2. Django Architecture — MVT (Model-View-Template)

> **📣 Definition:** _"Django uses MVT (Model-View-Template), which is its variant of the classic MVC pattern. **Model** = the data layer (ORM classes mapping to DB tables). **View** = the business logic that processes a request and returns a response (think 'controller' in MVC). **Template** = the presentation layer (HTML with template tags). Django's URL dispatcher routes requests to the right View. The framework is loosely coupled — each layer can be swapped or tested independently."_

#### 🧑‍🍳 Layman

Django follows **MVT** (not MVC). Imagine a restaurant:

- **Model** = the kitchen pantry (data + recipes)
- **View** = the chef (gets the order, picks ingredients from pantry, hands the dish to a waiter)
- **Template** = the plate presentation (how the food looks to the customer)

The "Controller" role (deciding which chef handles which order) is handled by Django itself via the URL dispatcher.

#### 🔧 Technical

```
Browser Request
    │
    ▼
WSGI/ASGI server (gunicorn/uvicorn)
    │
    ▼
Django middleware stack (request phase, top→bottom)
    │
    ▼
URL dispatcher (urls.py) → matches path to view
    │
    ▼
View (function or class)
    ├──► Model (ORM queries DB)
    └──► Template (renders HTML) OR JsonResponse
    │
    ▼
Django middleware stack (response phase, bottom→top)
    │
    ▼
Browser Response
```

**Project vs App distinction (interviewers love this):**

- **Project** = the entire web application (one `settings.py`, one `urls.py` root).
- **App** = a self-contained module with one focused responsibility (users, orders, payments). A project contains many apps. Apps should be reusable across projects.

#### 📌 TLDR

> "Django uses MVT — Model handles data, View handles business logic, Template handles presentation. The 'controller' role is the URL dispatcher, handled by the framework. A Django project is the entire app; apps are reusable, focused modules within it."

---

### 3. Django Request-Response Lifecycle (Critical for Tech Lead)

> **📣 Definition:** _"When a request arrives at a Django app, the WSGI/ASGI server passes it through the middleware chain (top→bottom for requests), the URL resolver matches the path to a view, the view executes business logic and queries the ORM, returns an HttpResponse, then the response passes back through middleware in reverse order before being sent to the browser. Knowing this order matters because it determines where you place auth, logging, security headers, and exception handlers."_

```
1. WSGI/ASGI server receives HTTP request
2. Request hits middleware chain (process_request, top → bottom)
3. URL resolver finds the matching view via urls.py
4. View-level middleware runs (process_view)
5. View executes — queries DB via ORM, processes data
6. View returns HttpResponse / TemplateResponse / JsonResponse
7. Template middleware runs if template needs rendering
8. Response middleware runs (bottom → top)
9. WSGI/ASGI sends response back to browser
```

**Interview-friendly diagram of middleware flow:**

```
Request →  [SecurityMiddleware]
        →  [SessionMiddleware]
        →  [AuthenticationMiddleware]
        →  [CSRFMiddleware]
        →  [YourCustomMiddleware]    ← request flows DOWN
        →     [VIEW EXECUTES]         ← view in the middle
        ←  [YourCustomMiddleware]    ← response flows UP
        ←  [CSRFMiddleware]
        ←  [AuthenticationMiddleware]
        ←  [SessionMiddleware]
        ←  [SecurityMiddleware]
Response ←
```

#### 📌 TLDR

> "Request flows top-to-bottom through middleware → URL resolver → view → response flows bottom-to-top through middleware. Order in `MIDDLEWARE` setting matters — security stuff goes outermost, app-specific stuff inner."

---

### 4. Models, Fields, and Relationships

> **📣 Definition:** _"Django models are Python classes that map to database tables — each attribute becomes a column. Field types (`CharField`, `IntegerField`, `DateTimeField`) define column types and validation. Three relationship types: `ForeignKey` (many-to-one, e.g. many books to one author), `OneToOneField` (one-to-one, like User → Profile), and `ManyToManyField` (creates a join table automatically). The `Meta` class controls table-level options like ordering, indexes, and unique constraints."_

```python
from django.db import models

class Category(models.Model):
    name = models.CharField(max_length=100, unique=True, db_index=True)
    slug = models.SlugField(unique=True)

    class Meta:
        verbose_name_plural = "Categories"
        ordering = ["name"]
        indexes = [models.Index(fields=["name"], name="cat_name_idx")]
        constraints = [
            models.CheckConstraint(check=~models.Q(name=""), name="non_empty_name")
        ]

    def __str__(self):
        return self.name


class Product(models.Model):
    STATUS_CHOICES = [
        ("draft", "Draft"),
        ("active", "Active"),
        ("archived", "Archived"),
    ]
    name = models.CharField(max_length=200)
    category = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,         # don't allow category deletion if products exist
        related_name="products",          # Category.products.all() instead of product_set
    )
    tags = models.ManyToManyField("Tag", related_name="products", blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default="draft")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

**Relationship types — quick reference:**

| Relationship | Field             | Reverse Access                             | DB Storage                      |
| ------------ | ----------------- | ------------------------------------------ | ------------------------------- |
| One-to-Many  | `ForeignKey`      | `parent.child_set.all()` or `related_name` | FK column on child table        |
| One-to-One   | `OneToOneField`   | `obj.related_obj` (single instance)        | Unique FK column on owner table |
| Many-to-Many | `ManyToManyField` | `obj.related_set.all()`                    | Separate join table             |

**`on_delete` behaviors (huge interview topic):**

| Option        | Behavior                                              | Use case                                           |
| ------------- | ----------------------------------------------------- | -------------------------------------------------- |
| `CASCADE`     | Delete dependent rows when parent is deleted          | Comments when a Post is deleted                    |
| `PROTECT`     | Raise `ProtectedError` to prevent parent deletion     | Don't let category be deleted if products exist    |
| `SET_NULL`    | Set FK to NULL (field must be `null=True`)            | Author of a blog post — keep post if author leaves |
| `SET_DEFAULT` | Set FK to a default value                             | Reassign to "Unknown" category                     |
| `SET()`       | Set to value/callable                                 | `SET(get_sentinel_user)` for soft-delete patterns  |
| `DO_NOTHING`  | Database handles it (rare, dangerous)                 | Custom triggers handle deletion                    |
| `RESTRICT`    | Like PROTECT, but allows cascading via other FK paths | More flexible than PROTECT                         |

#### 📌 TLDR

> "Use `ForeignKey` for one-to-many, `OneToOneField` for one-to-one, `ManyToManyField` for many-to-many (creates join table). `on_delete=CASCADE` for dependent data, `PROTECT` to prevent accidental deletion, `SET_NULL` to preserve history. Always set `related_name` for cleaner reverse lookups."

---

### 5. Model Inheritance — Three Strategies

> **📣 Definition:** _"Django offers three model inheritance patterns. **Abstract base classes** (`abstract = True`) — no table for the parent; fields are copied into children. Best for shared fields like `TimestampedModel`. **Multi-table inheritance** — separate table per class connected via OneToOne, gives true polymorphism but JOINs hurt performance. **Proxy models** — same table, different Python class with custom managers or methods, no schema change. The most common in production is abstract — use it for mixins."_

```python
# 1. ABSTRACT BASE CLASSES — no DB table for parent, fields inherited
class TimeStampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True   # NO table created for this!

class Article(TimeStampedModel):  # gets created_at, updated_at columns
    title = models.CharField(max_length=200)


# 2. MULTI-TABLE INHERITANCE — separate tables, one-to-one relationship
class Place(models.Model):
    name = models.CharField(max_length=50)
    address = models.CharField(max_length=80)

class Restaurant(Place):  # creates a `place_ptr` OneToOne to Place
    serves_pizza = models.BooleanField()

# Restaurant has its own table + auto FK to Place table
# Restaurant.objects.all() does an implicit JOIN — can be slow!


# 3. PROXY MODELS — same table, different Python behavior
class OrderedPerson(Person):
    class Meta:
        proxy = True
        ordering = ["last_name"]

    def get_full_name(self):  # add new behavior without new fields
        return f"{self.first_name} {self.last_name}"
```

**When to use which:**

| Strategy        | Use when                                                       | Avoid when                        |
| --------------- | -------------------------------------------------------------- | --------------------------------- |
| **Abstract**    | Sharing common fields (timestamps, soft-delete)                | Almost never bad — your default   |
| **Multi-table** | Real "is-a" hierarchy with extra fields per subtype            | Performance matters (extra JOINs) |
| **Proxy**       | Different default ordering, managers, or methods on same table | You need new fields               |

#### 📌 TLDR

> "Three inheritance strategies: **Abstract** (no parent table, fields inherited — most common, use for `TimeStampedModel`-style mixins), **Multi-table** (separate table + OneToOne, real polymorphism but JOINs hurt perf), **Proxy** (same table, different Python behavior — alternative ordering/managers)."

---

### 6. QuerySets — Lazy Evaluation & Chaining

> **📣 Definition:** _"A Django QuerySet is a lazy, chainable representation of a database query. Building it (`.filter()`, `.exclude()`, `.order_by()`) doesn't hit the database — only iterating, slicing, calling `len()` or `list()` triggers the actual SQL. This laziness lets you compose queries dynamically, but it also means a single ORM expression can hide expensive operations or trigger N+1 queries when chained with related field access."_

#### 🧑‍🍳 Layman

A QuerySet is like a **shopping list**, not the actual shopping trip. You add items (filters), reorganize, etc. — no money is spent yet. Only when you actually go to the store (iterate, call `len()`, slice, etc.) does it execute.

#### 🔧 Technical

```python
# These DON'T hit the DB — just building the SQL
qs = Product.objects.all()                    # no query
qs = qs.filter(status="active")               # no query
qs = qs.exclude(price__lt=100)                # no query
qs = qs.order_by("-created_at")               # no query
qs = qs[:20]                                  # no query (slicing)

# These DO hit the DB:
list(qs)                # iteration
qs[0]                   # indexing (with LIMIT)
len(qs)                 # forces evaluation (use .count() instead!)
bool(qs)                # forces evaluation (use .exists() instead!)
print(qs)               # __repr__ evaluates
for item in qs: ...     # iteration
```

**QuerySet caching:**

```python
qs = Product.objects.all()
list(qs)           # Query 1: SQL hits DB, results cached on the QuerySet
list(qs)           # NO query — uses cache

# But this is a NEW QuerySet, so NEW query:
list(Product.objects.all())   # Query 2
```

**Key methods:**

```python
Product.objects.count()              # SELECT COUNT(*) — much faster than len(qs)
Product.objects.exists()             # SELECT 1 ... LIMIT 1 — fastest existence check
Product.objects.first()              # First object or None (no exception)
Product.objects.last()               # Last object or None
Product.objects.get(id=5)            # Returns object OR raises DoesNotExist/MultipleObjectsReturned
Product.objects.values("id", "name") # Returns dict-like, much lighter than full objects
Product.objects.values_list("id", flat=True)   # Returns flat list of IDs
Product.objects.in_bulk([1, 2, 3])   # Returns {id: object} dict
Product.objects.distinct()           # SELECT DISTINCT
Product.objects.iterator()           # Memory-efficient streaming — for huge result sets
Product.objects.none()               # Empty QuerySet — useful in conditional logic
```

**`.get()` vs `.filter()` — The Most Asked Django ORM Question:**

```python
# === .get() — returns a SINGLE OBJECT or raises an exception ===
# SQL: SELECT ... WHERE id = 5 LIMIT 1

product = Product.objects.get(id=5)
# Returns: <Product: iPhone 15>              ← a single model instance
type(product)  # <class 'myapp.models.Product'>

# ⚠️ Raises DoesNotExist if NO match:
Product.objects.get(id=999)
# ❌ Product.DoesNotExist: Product matching query does not exist.

# ⚠️ Raises MultipleObjectsReturned if MORE THAN ONE match:
Product.objects.get(status="active")    # if multiple active products exist
# ❌ Product.MultipleObjectsReturned: get() returned more than one Product

# Safe pattern:
try:
    product = Product.objects.get(id=5)
except Product.DoesNotExist:
    product = None

# Or use shortcut:
from django.shortcuts import get_object_or_404
product = get_object_or_404(Product, id=5)    # returns object or raises Http404


# === .filter() — returns a QUERYSET (always, even if empty or single) ===
# SQL: SELECT ... WHERE status = 'active'

products = Product.objects.filter(status="active")
# Returns: <QuerySet [<Product: iPhone>, <Product: Pixel>]>   ← QuerySet, not object
type(products)  # <class 'django.db.models.query.QuerySet'>

# Empty result — no exception, just empty QuerySet:
products = Product.objects.filter(status="discontinued")
# Returns: <QuerySet []>                     ← empty, no error!
products.exists()   # False
len(products)       # 0

# Single match — still returns QuerySet:
products = Product.objects.filter(id=5)
# Returns: <QuerySet [<Product: iPhone 15>]>  ← QuerySet with 1 item, NOT the object

# To get the object from a filter:
product = Product.objects.filter(id=5).first()   # returns object or None (no exception)
```

**Side-by-side comparison:**

| Aspect               | `.get()`                                   | `.filter()`                               |
| -------------------- | ------------------------------------------ | ----------------------------------------- |
| **Returns**          | Single model instance                      | QuerySet (list-like)                      |
| **No match**         | ❌ Raises `DoesNotExist`                   | ✅ Returns empty QuerySet `[]`            |
| **Multiple matches** | ❌ Raises `MultipleObjectsReturned`        | ✅ Returns all matches                    |
| **Chainable**        | ❌ No (it's a terminal call)               | ✅ Yes (`.filter().exclude().order_by()`) |
| **Lazy**             | ❌ No (hits DB immediately)                | ✅ Yes (hits DB only when evaluated)      |
| **Use when**         | Need exactly 1 record (by PK/unique field) | Need 0, 1, or many records                |
| **Safe alternative** | `get_object_or_404()` / try-except         | `.first()` (returns None if empty)        |

```python
# === Common Interview Follow-ups ===

# Q: "When should you use .get() vs .filter().first()?"
Product.objects.get(id=5)              # raises if not found — use when absence is an ERROR
Product.objects.filter(id=5).first()   # returns None if not found — use when absence is OK

# Q: "What SQL does each generate?"
Product.objects.get(id=5)
# SELECT * FROM product WHERE id = 5     (no LIMIT — Django checks count in Python)

Product.objects.filter(id=5)
# No SQL yet! (lazy)

Product.objects.filter(id=5).first()
# SELECT * FROM product WHERE id = 5 LIMIT 1

# Q: "Can you chain after .get()?"
Product.objects.get(id=5).filter(...)    # ❌ AttributeError — .get() returns an object, not QS
Product.objects.filter(id=5).exclude(...)  # ✅ Works — .filter() returns a QuerySet
```

**Field lookups:**

```python
Product.objects.filter(name__exact="Phone")        # case-sensitive
Product.objects.filter(name__iexact="phone")       # case-insensitive
Product.objects.filter(name__contains="pro")       # LIKE %pro%
Product.objects.filter(name__icontains="PRO")      # case-insensitive contains
Product.objects.filter(name__startswith="iPhone")
Product.objects.filter(price__gte=100, price__lte=500)
Product.objects.filter(category__name="Electronics")  # JOIN via FK
Product.objects.filter(tags__name__in=["sale", "new"]) # M2M lookup
Product.objects.filter(created_at__year=2026)
Product.objects.filter(created_at__date=date(2026, 4, 27))
Product.objects.filter(name__regex=r"^iPhone\s\d+")
Product.objects.filter(metadata__color="red")      # JSONField lookup
Product.objects.filter(name__isnull=True)
```

#### 📌 TLDR

> "QuerySets are lazy — they build SQL incrementally and only execute when consumed (iterated, sliced with index, `len()`, `bool()`, etc.). Use `.exists()` instead of `if qs:` and `.count()` instead of `len(qs)`. Once evaluated, results are cached on the QuerySet. Use `.iterator()` for huge result sets to avoid loading everything into memory."

---

### 7. `Q` Objects — Complex Query Logic (OR/AND/NOT)

> **📣 Definition:** _"By default, multiple keyword filters in Django are AND-ed together. `Q` objects let you build OR (`|`), NOT (`~`), and complex parenthesized boolean expressions in a single query. Without them, you'd be stuck with `filter(...)` (AND) or chaining multiple QuerySets (which can't easily express OR). Essential for search, multi-criteria filtering, and any 'either/or' logic in the WHERE clause."_

#### 🧑‍🍳 Layman

By default, `filter(a=1, b=2)` means "a AND b". But what if you want "a OR b"? Or "NOT a"? That's where `Q` objects come in — they let you build complex boolean expressions.

#### 🔧 Technical

```python
from django.db.models import Q

# OR — find products that are either active OR featured
Product.objects.filter(Q(status="active") | Q(is_featured=True))

# AND (explicit) — same as filter(price__gte=100, status="active")
Product.objects.filter(Q(price__gte=100) & Q(status="active"))

# NOT — exclude active products
Product.objects.filter(~Q(status="active"))

# Complex combinations — (active AND price>100) OR featured
Product.objects.filter(
    (Q(status="active") & Q(price__gte=100)) | Q(is_featured=True)
)

# Mixing Q with kwargs — Q objects must come BEFORE kwargs
Product.objects.filter(
    Q(name__icontains="phone") | Q(name__icontains="laptop"),
    status="active",   # AND combined with the Q expression
)
```

**Real-world: Dynamic search across multiple fields**

```python
from functools import reduce
from operator import or_

def search_products(query, search_fields=("name", "description", "sku")):
    if not query:
        return Product.objects.all()

    # Build OR query across all search fields dynamically
    q_objects = [Q(**{f"{field}__icontains": query}) for field in search_fields]
    combined = reduce(or_, q_objects)

    return Product.objects.filter(combined)

# Usage: search "phone" in name, description, AND sku
results = search_products("phone")
```

#### 📌 TLDR

> "`Q` objects let you combine query conditions with `|` (OR), `&` (AND), `~` (NOT). Essential for dynamic searches, permission-based filtering, and any query that goes beyond simple AND. Use `reduce(or_, q_objects)` for dynamic multi-field search — a pattern that scales when search fields change."

---

### 8. `F` Expressions — Atomic Updates & Field-to-Field Comparison

> **📣 Definition:** _"`F` expressions reference a database field directly within a query, letting Django perform updates at the SQL level rather than in Python. This makes operations atomic and avoids race conditions: `Product.objects.filter(id=1).update(stock=F('stock') - 1)` decrements stock in a single SQL statement, immune to concurrent updates. `F` is also used for comparing two fields on the same row in a filter, like 'find products where price > cost'."_

#### 🧑‍🍳 Layman

Without `F`, to increment a counter you'd: read value → add 1 → write back. Two users doing this at once = lost update! `F` says "do this math AT THE DATABASE LEVEL" — atomic, safe.

#### 🔧 Technical

```python
from django.db.models import F

# WRONG — race condition!
product = Product.objects.get(id=1)
product.stock = product.stock - 1   # read in Python
product.save()                       # write back — but another request might've changed it!

# RIGHT — atomic database-level update
Product.objects.filter(id=1).update(stock=F("stock") - 1)
# SQL: UPDATE product SET stock = stock - 1 WHERE id = 1
# This is atomic — two concurrent requests can't clobber each other

# Field-to-field comparison
Product.objects.filter(stock__lt=F("min_stock_threshold"))
# Find products where stock is below their own threshold

# Using F in annotate
from django.db.models import F
products = Product.objects.annotate(profit=F("selling_price") - F("cost_price"))

# F with related fields (via __)
Order.objects.filter(amount__gt=F("user__credit_limit"))
```

#### 📌 TLDR

> "`F` expressions reference column values at the DB level. Use them for atomic updates (counters, stock decrements) — avoids race conditions where two requests read-modify-write the same row. Also useful for field-to-field comparisons in filters."

---

### 9. `select_related` vs `prefetch_related` — The N+1 Killer

> **📣 Definition:** _"Both eliminate N+1 queries when accessing related objects, but they work differently. `select_related` performs a SQL JOIN — use it for ForeignKey and OneToOneField (single-valued relationships, 'parent' direction). `prefetch_related` runs a separate query and stitches results in Python — use it for ManyToManyField and reverse ForeignKeys (many-valued relationships, 'children' direction). Without these, accessing related objects in a loop turns one query into hundreds. This is the #1 Django interview question at any level."_

This is probably the **#1 most-asked Django interview question** at any level.

#### 🧑‍🍳 Layman

**The N+1 problem**: You fetch 100 books from the DB (1 query). Then for each book, you access `book.author` — that's 100 MORE queries. Total: 101 queries for what should be 1-2.

`select_related` and `prefetch_related` are the solutions — they pre-fetch related data upfront so accessing it doesn't trigger more queries.

#### 🔧 Technical

**The N+1 problem demonstrated:**

```python
# Models
class Author(models.Model):
    name = models.CharField(max_length=100)

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.ForeignKey(Author, on_delete=models.CASCADE, related_name="books")
    tags = models.ManyToManyField("Tag", related_name="books")

# THE PROBLEM (N+1 queries):
books = Book.objects.all()           # Query 1: SELECT * FROM book
for book in books:
    print(book.author.name)          # Query 2, 3, 4, ... N+1: SELECT * FROM author WHERE id = ?
# 100 books → 101 queries total. Production-killer.
```

**`select_related` — for ForeignKey & OneToOne (forward relationships):**

```python
# Uses a SQL JOIN — single query
books = Book.objects.select_related("author").all()
for book in books:
    print(book.author.name)          # NO additional query — already loaded

# SQL generated:
# SELECT book.*, author.* FROM book INNER JOIN author ON book.author_id = author.id

# Multi-level — chain with __
Book.objects.select_related("author__profile__country")
# JOINs book → author → profile → country in ONE query
```

**`prefetch_related` — for ManyToMany & reverse ForeignKey:**

```python
# Cannot use JOIN (would multiply rows). Uses 2 queries + Python merge.
books = Book.objects.prefetch_related("tags").all()
for book in books:
    for tag in book.tags.all():       # NO additional queries
        print(tag.name)

# SQL generated:
# Query 1: SELECT * FROM book
# Query 2: SELECT * FROM tag INNER JOIN book_tag ON ... WHERE book_id IN (1, 2, 3, ...)
# Django merges the results in Python

# Reverse ForeignKey example — get authors with their books
authors = Author.objects.prefetch_related("books").all()
for author in authors:
    for book in author.books.all():    # NO query — prefetched
        print(book.title)
```

**Decision matrix:**

| Relationship                               | Method               | Why                                              |
| ------------------------------------------ | -------------------- | ------------------------------------------------ |
| `ForeignKey` (forward) — `book.author`     | `select_related`     | One author per book → JOIN works                 |
| `OneToOneField` (forward) — `user.profile` | `select_related`     | One-to-one → JOIN works                          |
| `ForeignKey` (reverse) — `author.books`    | `prefetch_related`   | Many books per author → JOIN would multiply rows |
| `ManyToManyField` — `book.tags`            | `prefetch_related`   | Many-to-many → can't JOIN sensibly               |
| **Mixing them**                            | **Combine in chain** | See below                                        |

**Combining both:**

```python
# Order has FK to Customer (forward) and reverse FK to OrderItem
orders = Order.objects.select_related("customer").prefetch_related("items").all()

# Going further — items have FK to product
orders = Order.objects.select_related("customer").prefetch_related("items__product")
# items__product = prefetch items, then for each item, prefetch its product
```

**`Prefetch` object — for filtering/ordering prefetched data:**

```python
from django.db.models import Prefetch

# Get authors with only their RECENT books, ordered by date
authors = Author.objects.prefetch_related(
    Prefetch(
        "books",
        queryset=Book.objects.filter(published_year__gte=2025).order_by("-published_year"),
        to_attr="recent_books",   # store in custom attribute (not 'books')
    )
)

for author in authors:
    for book in author.recent_books:    # use the custom attribute
        print(book.title)
```

**Encode optimizations in custom managers (tech-lead pattern):**

```python
class BookQuerySet(models.QuerySet):
    def with_author(self):
        return self.select_related("author")

    def with_full_details(self):
        return self.select_related("author", "publisher").prefetch_related(
            "tags",
            Prefetch(
                "reviews",
                queryset=Review.objects.select_related("user").order_by("-created_at"),
            ),
        )

class Book(models.Model):
    # ... fields ...
    objects = BookQuerySet.as_manager()

# Now in views:
Book.objects.with_full_details()   # Self-documenting, optimization in one place
```

**Detection — Django Debug Toolbar + assertNumQueries:**

```python
# In tests — prevent regressions
class BookListTest(TestCase):
    def test_query_count(self):
        Author.objects.create(name="Test")
        for i in range(100):
            Book.objects.create(title=f"Book {i}", author_id=1)

        # Assert exactly 2 queries (not 101)
        with self.assertNumQueries(2):
            response = self.client.get("/books/")
            list(response.context["books"])  # force eval
```

#### 📌 TLDR

> "N+1 is the #1 Django performance killer. **`select_related`** uses SQL JOIN, works for `ForeignKey`/`OneToOne` (forward). **`prefetch_related`** uses 2 queries + Python merge, works for `ManyToMany` and reverse `ForeignKey`. Combine them in a chain. Use `Prefetch` objects for filtering prefetched data. Encode common optimizations in custom managers (e.g., `Book.objects.with_full_details()`). Test with `assertNumQueries` to prevent regressions."

---

### 10. Aggregation, Annotation & Subqueries

> **📣 Definition:** _"`aggregate()` returns a single dict of computed values across the entire QuerySet (`SUM`, `AVG`, `COUNT` etc.). `annotate()` attaches computed fields to each row in the result — like adding a `book_count` to each author. `Subquery` + `OuterRef` lets you use a query inside another query for things SQL JOINs can't easily express, like 'each author's most recent book title'. Together these replace the need for raw SQL in most reporting and analytics queries."_

```python
from django.db.models import Count, Sum, Avg, Max, Min, F, Q, Subquery, OuterRef

# AGGREGATE — single value across the queryset
Product.objects.aggregate(
    total_revenue=Sum("price"),
    avg_price=Avg("price"),
    product_count=Count("id"),
)
# Returns: {"total_revenue": 1500000, "avg_price": 5000, "product_count": 300}

# ANNOTATE — add a computed field to EACH row
authors = Author.objects.annotate(book_count=Count("books"))
for author in authors:
    print(author.name, author.book_count)   # uses annotated field

# Conditional annotation
from django.db.models import Case, When, Value, IntegerField

Order.objects.annotate(
    priority=Case(
        When(amount__gte=10000, then=Value(1)),
        When(amount__gte=1000, then=Value(2)),
        default=Value(3),
        output_field=IntegerField(),
    )
)

# SUBQUERY — query within a query
latest_book = Book.objects.filter(author=OuterRef("pk")).order_by("-published_at")
authors = Author.objects.annotate(
    latest_book_title=Subquery(latest_book.values("title")[:1])
)

# EXISTS subquery — much faster for "does X exist" patterns
from django.db.models import Exists
authors_with_recent_books = Author.objects.annotate(
    has_recent=Exists(Book.objects.filter(author=OuterRef("pk"), published_year__gte=2025))
).filter(has_recent=True)
```

#### 📌 TLDR

> "**`aggregate`** returns one value across the whole queryset. **`annotate`** adds a computed field to each row (uses `GROUP BY`). Use `Subquery`/`OuterRef` for correlated subqueries (e.g., 'each author's latest book'). Use `Exists` instead of `Count() > 0` — it's much faster because the DB short-circuits."

---

### 11. Custom Managers & QuerySets

> **📣 Definition:** _"A custom Manager is the entry point to the ORM for a model — `Model.objects` is the default. By overriding `get_queryset()` you can create scoped managers like `Product.active.all()` that pre-filter to active rows. Custom QuerySet methods let you chain domain-specific filters (`.in_stock().on_sale()`). Together they keep query logic out of views and into the model layer, where it belongs — Django's version of the Repository pattern."_

```python
class ActiveProductManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().filter(status="active")

class Product(models.Model):
    name = models.CharField(max_length=100)
    status = models.CharField(max_length=20)

    objects = models.Manager()              # default — all rows
    active = ActiveProductManager()         # custom — only active

# Usage:
Product.objects.all()      # all products
Product.active.all()       # only active products

# BETTER PATTERN: Custom QuerySet with chainable methods
class ProductQuerySet(models.QuerySet):
    def active(self):
        return self.filter(status="active")

    def in_category(self, category):
        return self.filter(category=category)

    def cheap(self, threshold=500):
        return self.filter(price__lt=threshold)

class Product(models.Model):
    # ... fields ...
    objects = ProductQuerySet.as_manager()

# Now you can chain:
Product.objects.active().in_category(electronics).cheap()
# Self-documenting + reusable
```

#### 📌 TLDR

> "Use `Manager` to encapsulate default filtering (`Product.active.all()`). Better: define a custom `QuerySet` and expose it via `.as_manager()` — methods become chainable. This is how production Django codebases stay clean."

---

### 12. Signals — Power & Pitfalls

> **📣 Definition:** _"Django signals are an Observer-pattern implementation that lets decoupled parts of an app react to events — like 'a User was saved' (`post_save`) — without explicit coupling. They're powerful but dangerous: signals create implicit, hard-to-trace dependencies, run synchronously by default (slowing down saves), and can fire unexpectedly in tests or migrations. The senior take: prefer explicit service methods over signals; reserve signals for truly cross-cutting concerns where the sender shouldn't know about the receiver."_

#### 🧑‍🍳 Layman

Signals are like **event listeners**. When something happens (e.g., a User is saved), a signal fires, and any function connected to that signal automatically runs. Useful for cross-cutting concerns like "auto-create a profile when a user is registered."

#### 🔧 Technical

```python
from django.db.models.signals import (
    pre_save, post_save, pre_delete, post_delete, m2m_changed
)
from django.dispatch import receiver
from django.contrib.auth.models import User

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:   # only on creation, not on update
        Profile.objects.create(user=instance)

# Custom signal
from django.dispatch import Signal
order_completed = Signal()  # define

# Send the signal
order_completed.send(sender=Order, order=order_instance, total=500)

# Receive it
@receiver(order_completed)
def send_order_email(sender, order, total, **kwargs):
    send_email(order.customer.email, f"Your order ₹{total} is confirmed!")
```

**Common built-in signals:**

| Signal                                 | When fires                                   |
| -------------------------------------- | -------------------------------------------- |
| `pre_save` / `post_save`               | Before/after `Model.save()`                  |
| `pre_delete` / `post_delete`           | Before/after `Model.delete()`                |
| `m2m_changed`                          | M2M relationship modified (add/remove/clear) |
| `request_started` / `request_finished` | Request lifecycle                            |
| `user_logged_in` / `user_logged_out`   | Auth events                                  |

**Pitfalls (interview-level):**

1. **Signals are SYNCHRONOUS by default** — they run in the same request/transaction. A slow signal handler slows the entire request.

   ```python
   @receiver(post_save, sender=Order)
   def send_email_on_order(sender, instance, **kwargs):
       send_email(instance.customer.email, ...)   # ❌ blocks the response!
   # Fix: dispatch to Celery: send_email.delay(...)
   ```

2. **Hidden side effects** — signals run "magically" from anywhere. Hard to debug, hard to test, breaks "explicit is better than implicit."

   ```python
   # Calling Order.objects.create() can trigger 5 signals across the codebase
   # New developer has no idea what's happening
   ```

3. **`bulk_create` does NOT fire signals!** Major gotcha.

   ```python
   Order.objects.bulk_create([Order(...), Order(...)])  # NO post_save fired!
   # Only individual .save() fires signals
   ```

4. **`raw=True` in `pre_save`/`post_save`** — fires during fixture loading; check this flag to skip side effects.

   ```python
   @receiver(post_save, sender=Order)
   def handler(sender, instance, raw, **kwargs):
       if raw:
           return    # don't run during loaddata fixtures
       # ... actual logic
   ```

5. **`update_fields` parameter** — check what changed, avoid unnecessary work.

   ```python
   @receiver(post_save, sender=User)
   def handler(sender, instance, update_fields, **kwargs):
       if update_fields and "email" in update_fields:
           send_email_changed_notification(instance)
   ```

6. **Disconnecting signals** (e.g., in tests):

   ```python
   from django.db.models.signals import post_save

   post_save.disconnect(create_user_profile, sender=User)
   # ... do stuff ...
   post_save.connect(create_user_profile, sender=User)

   # Or as context manager (using factory_boy / django-disable-signals)
   ```

7. **Use `transaction.on_commit` for side effects that need DB consistency**:

   ```python
   from django.db import transaction

   @receiver(post_save, sender=Order)
   def send_email(sender, instance, created, **kwargs):
       if created:
           transaction.on_commit(
               lambda: email_task.delay(instance.id)
           )
       # email only fires if the transaction commits successfully
   ```

**When NOT to use signals (tech-lead opinion):**

- When the calling code already knows what it wants — just call the function explicitly.
- Cross-app coupling that's better handled via dependency injection or service objects.
- Complex orchestration — use Celery + explicit task chains instead.

#### 📌 TLDR

> "Signals decouple side effects from save/delete via `post_save`, `pre_delete`, `m2m_changed`, etc. Powerful but dangerous: they're synchronous, magical, and `bulk_create` doesn't fire them. Use `transaction.on_commit()` to defer side effects until the txn succeeds. Senior devs prefer explicit service-layer calls over signals for non-trivial logic."

---

### 13. Middleware — Architecture & Custom Implementation

> **📣 Definition:** _"Middleware is Django's request/response processing pipeline — each component sees every request before it hits the view and every response before it leaves. The order in `MIDDLEWARE` settings matters: requests flow top→bottom, responses flow bottom→top. Use middleware for cross-cutting concerns: auth, security headers, request logging, CORS, exception handling. Custom middleware in modern Django is a class with `__init__(get_response)` and `__call__(request)`, optionally with `process_view`/`process_exception` hooks."_

#### 🧑‍🍳 Layman

Middleware is a **pipeline of filters** every request and response flows through. Each middleware can read/modify the request before the view runs, and read/modify the response before it's sent.

Like airport security: you go through metal detector → x-ray → boarding pass check → gate. Each is a "middleware."

#### 🔧 Technical

```python
# Modern Django middleware (1.10+)
class TimingMiddleware:
    def __init__(self, get_response):
        # One-time setup at server startup
        self.get_response = get_response

    def __call__(self, request):
        # === BEFORE the view ===
        import time
        start = time.time()

        # Call the next middleware / view
        response = self.get_response(request)

        # === AFTER the view ===
        duration = time.time() - start
        response["X-Response-Time"] = f"{duration:.3f}s"
        return response

# Optional hooks
class FullMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        return response

    def process_view(self, request, view_func, view_args, view_kwargs):
        """Called just BEFORE the view executes. Return None to continue,
        or return an HttpResponse to short-circuit."""
        pass

    def process_exception(self, request, exception):
        """Called when a view raises an exception. Return None to let Django
        handle it, or return an HttpResponse to handle it yourself."""
        pass

    def process_template_response(self, request, response):
        """Called for TemplateResponse objects after the view."""
        return response
```

**Register in `settings.py`:**

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "myapp.middleware.TimingMiddleware",   # custom
]
# ORDER MATTERS — security/auth outermost, app-specific innermost
```

**Real-world middleware examples:**

```python
# 1. Request ID for tracing
import uuid

class RequestIDMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request.id = request.META.get("HTTP_X_REQUEST_ID", str(uuid.uuid4()))
        response = self.get_response(request)
        response["X-Request-ID"] = request.id
        return response


# 2. Multi-tenancy
class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        host = request.get_host().split(".")[0]   # e.g., "acme.example.com"
        try:
            request.tenant = Tenant.objects.get(subdomain=host)
        except Tenant.DoesNotExist:
            return HttpResponseNotFound("Tenant not found")
        return self.get_response(request)


# 3. Conditional middleware
class FeatureFlagMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        if request.path.startswith("/api/v2/"):
            request.use_v2_serializers = True
        return self.get_response(request)
```

#### 📌 TLDR

> "Middleware is a pipeline: request flows top→bottom, view runs, response flows bottom→top. Implement `__init__(get_response)` and `__call__(request)`. Optional hooks: `process_view`, `process_exception`, `process_template_response`. Use for cross-cutting concerns: auth, logging, request IDs, multi-tenancy, rate limiting. Order in `MIDDLEWARE` matters — security outermost, app-specific innermost."

---

### 14. Class-Based Views (CBVs) vs Function-Based Views (FBVs)

> **📣 Definition:** _"FBVs are plain Python functions that take a request and return a response — explicit, easy to read, but verbose for repetitive CRUD. CBVs are classes that group related logic and inherit from generic views (`ListView`, `DetailView`, `CreateView`, etc.) — concise for standard patterns but with a steeper learning curve and hidden method-resolution-order behavior. Rule of thumb: CBVs for boilerplate CRUD, FBVs for unique logic where clarity beats DRY. Mixins like `LoginRequiredMixin` add reusable behavior to CBVs."_

```python
# FBV — explicit, simple
from django.shortcuts import render, get_object_or_404

def product_detail(request, pk):
    product = get_object_or_404(Product, pk=pk)
    return render(request, "product_detail.html", {"product": product})


# CBV — reusable, DRY, but more "magic"
from django.views.generic import DetailView, ListView, CreateView, UpdateView

class ProductDetailView(DetailView):
    model = Product
    template_name = "product_detail.html"
    context_object_name = "product"


# CBV with mixins
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin

class ProductCreateView(LoginRequiredMixin, PermissionRequiredMixin, CreateView):
    model = Product
    fields = ["name", "price", "category"]
    permission_required = "products.add_product"

    def form_valid(self, form):
        form.instance.created_by = self.request.user
        return super().form_valid(form)
```

**CBV vs FBV decision:**

| Use FBV when                        | Use CBV when                                     |
| ----------------------------------- | ------------------------------------------------ |
| Simple, one-off view                | Standard CRUD operations                         |
| Logic doesn't fit standard patterns | You can leverage generic views                   |
| You value explicit > implicit       | You want to share behavior via mixins            |
| Easier to understand for juniors    | Working in a large team with consistent patterns |

**Common generic CBVs:** `View`, `TemplateView`, `RedirectView`, `DetailView`, `ListView`, `CreateView`, `UpdateView`, `DeleteView`, `FormView`.

#### 📌 TLDR

> "FBVs are explicit and simple — easy to read. CBVs are DRY and reusable via inheritance/mixins (`LoginRequiredMixin`, `PermissionRequiredMixin`). Use generic CBVs for standard CRUD; use FBVs for one-off custom logic. In a tech-lead role, advocate for consistency — pick one as default for the team."

---

### 15. Django REST Framework (DRF) — Production Patterns

> **📣 Definition:** _"Django REST Framework is the de-facto standard for building REST APIs on Django. Core building blocks: **Serializers** convert between models and JSON (with validation), **ViewSets** group related endpoints (list, retrieve, create, update, destroy), **Routers** auto-generate URL patterns, **Permissions** control access, and **Throttling** for rate limiting. Production patterns include separate read/write serializers, nested serializers with `select_related`, custom permissions per action, and pagination classes. The lighter alternative is `django-ninja` (FastAPI-style for Django)."_

```python
# serializers.py
from rest_framework import serializers

class CategorySerializer(serializers.ModelSerializer):
    class Meta:
        model = Category
        fields = ["id", "name", "slug"]

class ProductSerializer(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    category_id = serializers.PrimaryKeyRelatedField(
        queryset=Category.objects.all(), source="category", write_only=True
    )
    profit_margin = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = ["id", "name", "price", "category", "category_id", "profit_margin"]

    def get_profit_margin(self, obj):
        return float(obj.price - obj.cost) if obj.cost else None

    def validate_price(self, value):
        if value < 0:
            raise serializers.ValidationError("Price cannot be negative")
        return value

    def validate(self, data):  # cross-field validation
        if data.get("price", 0) < data.get("cost", 0):
            raise serializers.ValidationError("Price must be >= cost")
        return data


# views.py — ViewSets are the standard
from rest_framework import viewsets, permissions, filters
from django_filters.rest_framework import DjangoFilterBackend

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.select_related("category").prefetch_related("tags")
    serializer_class = ProductSerializer
    permission_classes = [permissions.IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ["category", "status"]
    search_fields = ["name", "description"]
    ordering_fields = ["price", "created_at"]

    def get_queryset(self):
        # Override for custom filtering
        qs = super().get_queryset()
        if self.action == "list":
            qs = qs.filter(status="active")
        return qs

    @action(detail=True, methods=["post"])
    def archive(self, request, pk=None):
        # Custom action: POST /products/{id}/archive/
        product = self.get_object()
        product.status = "archived"
        product.save()
        return Response({"status": "archived"})


# urls.py
from rest_framework.routers import DefaultRouter
router = DefaultRouter()
router.register(r"products", ProductViewSet)
urlpatterns = router.urls
# Auto-generates: GET /products/, POST /products/, GET /products/{id}/, etc.
```

**Authentication options:**

- **TokenAuthentication** (`rest_framework.authtoken`) — DB-backed tokens
- **JWT** (via `djangorestframework-simplejwt`) — stateless, scalable
- **OAuth2** (via `django-oauth-toolkit`) — third-party access
- **SessionAuthentication** — for browser-based apps

**Permissions hierarchy:**

- `AllowAny` → no auth needed
- `IsAuthenticated` → must be logged in
- `IsAdminUser` → must be staff
- `IsAuthenticatedOrReadOnly` → reads open, writes need auth
- `DjangoModelPermissions` → uses Django's built-in permission system
- Custom: subclass `BasePermission`, override `has_permission` and `has_object_permission`

**Throttling (rate limiting):**

```python
# settings.py
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.UserRateThrottle",
        "rest_framework.throttling.AnonRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "user": "1000/day",
        "anon": "100/day",
    },
}
```

#### 📌 TLDR

> "DRF: serializers handle validation + Python ↔ JSON, ViewSets bundle CRUD into one class, routers auto-generate URLs. Use `select_related`/`prefetch_related` in `get_queryset()` to avoid N+1. Authentication: JWT for stateless APIs. Permissions: built-ins or custom `BasePermission` subclasses. Throttle by user/anon."

---

### 16. DRF Serializers — Deep Dive

> **📣 Definition:** _"A serializer converts complex data (Django model instances, querysets) into Python primitives that can be rendered into JSON/XML, and also **deserializes** incoming JSON back into validated Python objects. Think of it as Django Forms for APIs — it handles validation, field-level control, nested relationships, and read vs write representations. `Serializer` is the base (explicit fields); `ModelSerializer` auto-generates fields from a model. Knowing when to use which, how validation flows, and how to handle nested writes are the top interview questions."_

**`Serializer` vs `ModelSerializer`:**

```python
from rest_framework import serializers

# === Base Serializer — full control, no model coupling ===
class LoginSerializer(serializers.Serializer):
    """Use for non-model data (login, password reset, webhooks, etc.)."""
    email = serializers.EmailField()
    password = serializers.CharField(write_only=True, min_length=8)
    remember_me = serializers.BooleanField(default=False)

    def validate_email(self, value):
        return value.lower()

    def validate(self, data):
        # Cross-field validation
        user = authenticate(email=data["email"], password=data["password"])
        if not user:
            raise serializers.ValidationError("Invalid credentials")
        data["user"] = user
        return data


# === ModelSerializer — auto-generates fields from model ===
class UserSerializer(serializers.ModelSerializer):
    """Use when serializing/deserializing a Django model."""
    class Meta:
        model = User
        fields = ["id", "email", "first_name", "last_name", "date_joined"]
        read_only_fields = ["id", "date_joined"]
        extra_kwargs = {
            "email": {"required": True},
            "first_name": {"min_length": 2},
        }

# What ModelSerializer auto-generates for you:
# - Fields matching model field types (CharField, IntegerField, etc.)
# - Validators from model constraints (unique, max_length, etc.)
# - create() and update() methods that call model.save()
```

**Field Types & Common Kwargs:**

| Field                         | Use For      | Key Kwargs                                    |
| ----------------------------- | ------------ | --------------------------------------------- |
| `CharField`                   | Text         | `max_length`, `min_length`, `trim_whitespace` |
| `IntegerField`                | Integers     | `min_value`, `max_value`                      |
| `FloatField` / `DecimalField` | Numbers      | `max_digits`, `decimal_places`                |
| `BooleanField`                | True/False   | —                                             |
| `DateTimeField`               | Timestamps   | `format`, `input_formats`                     |
| `EmailField`                  | Emails       | Auto-validates email format                   |
| `FileField` / `ImageField`    | Uploads      | `max_length`, `use_url`                       |
| `ListField`                   | JSON arrays  | `child=serializers.IntegerField()`            |
| `DictField`                   | JSON objects | `child=serializers.CharField()`               |
| `SlugRelatedField`            | FK by slug   | `slug_field="username"`, `queryset=...`       |
| `PrimaryKeyRelatedField`      | FK by PK     | `queryset=...`, `many=True`                   |
| `SerializerMethodField`       | Computed     | Calls `get_<field_name>(self, obj)`           |
| `HiddenField`                 | Auto-set     | `default=CurrentUserDefault()`                |

**Universal kwargs:** `read_only`, `write_only`, `required`, `default`, `allow_null`, `source`, `validators`

**Validation — The 3 Layers (execution order):**

```python
class OrderSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["id", "product", "quantity", "total", "status"]

    # LAYER 1: Field-level validation — runs per-field
    def validate_quantity(self, value):
        """Called automatically for the 'quantity' field."""
        if value <= 0:
            raise serializers.ValidationError("Quantity must be positive")
        if value > 1000:
            raise serializers.ValidationError("Max 1000 per order")
        return value

    # LAYER 2: Object-level validation — runs after all fields pass
    def validate(self, data):
        """Cross-field validation. Receives ALL validated field data."""
        if data.get("status") == "shipped" and not data.get("tracking_number"):
            raise serializers.ValidationError(
                {"tracking_number": "Required when status is 'shipped'"}
            )
        return data

    # LAYER 3: Model-level validation — runs on model.full_clean()
    # (only if you call it in create/update — NOT automatic in DRF!)

# Validation execution order:
# 1. Field deserialization (type coercion)
# 2. Field validators (field-level validate_<field>)
# 3. Object validators (validate method)
# 4. UniqueTogetherValidator, UniqueForDateValidator (auto from model)
```

**Nested Serializers — Read vs Write Split (production pattern):**

```python
# === The Problem: nested data looks different on read vs write ===
# GET /orders/1/ → want full nested product: {"product": {"id": 1, "name": "Laptop"}}
# POST /orders/  → want to send just an ID: {"product_id": 1, "quantity": 2}

# === Solution 1: Separate Read/Write serializers (cleanest) ===
class OrderReadSerializer(serializers.ModelSerializer):
    product = ProductSerializer(read_only=True)        # nested object on read
    customer = CustomerSerializer(read_only=True)

    class Meta:
        model = Order
        fields = ["id", "product", "customer", "quantity", "total", "created_at"]

class OrderWriteSerializer(serializers.ModelSerializer):
    class Meta:
        model = Order
        fields = ["id", "product", "quantity"]         # product = FK (PK integer)

# In the ViewSet:
class OrderViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.action in ("list", "retrieve"):
            return OrderReadSerializer
        return OrderWriteSerializer


# === Solution 2: Same serializer, read_only + write_only fields ===
class OrderSerializer(serializers.ModelSerializer):
    product = ProductSerializer(read_only=True)        # response only
    product_id = serializers.PrimaryKeyRelatedField(
        queryset=Product.objects.all(),
        source="product",                              # maps to model's `product` FK
        write_only=True,                               # request only
    )

    class Meta:
        model = Order
        fields = ["id", "product", "product_id", "quantity", "total"]
```

**Writable Nested Serializers (creating related objects):**

```python
class AddressSerializer(serializers.ModelSerializer):
    class Meta:
        model = Address
        fields = ["street", "city", "zipcode"]

class CustomerSerializer(serializers.ModelSerializer):
    addresses = AddressSerializer(many=True)           # nested writable

    class Meta:
        model = Customer
        fields = ["id", "name", "email", "addresses"]

    def create(self, validated_data):
        addresses_data = validated_data.pop("addresses")
        customer = Customer.objects.create(**validated_data)
        for addr in addresses_data:
            Address.objects.create(customer=customer, **addr)
        return customer

    def update(self, instance, validated_data):
        addresses_data = validated_data.pop("addresses", None)
        instance.name = validated_data.get("name", instance.name)
        instance.email = validated_data.get("email", instance.email)
        instance.save()

        if addresses_data is not None:
            instance.addresses.all().delete()          # replace strategy
            for addr in addresses_data:
                Address.objects.create(customer=instance, **addr)
        return instance

# POST /customers/
# {"name": "Tushar", "email": "t@x.com", "addresses": [{"street": "...", "city": "..."}]}
```

**`source` kwarg — the hidden power tool:**

```python
class UserSerializer(serializers.ModelSerializer):
    # Rename fields
    full_name = serializers.CharField(source="get_full_name", read_only=True)   # calls method
    company = serializers.CharField(source="profile.company")                    # traverses relation
    role = serializers.CharField(source="get_role_display")                      # choices display

    # Nested dotted source
    department_name = serializers.CharField(source="department.name", read_only=True)

    class Meta:
        model = User
        fields = ["id", "email", "full_name", "company", "role", "department_name"]
```

**`SerializerMethodField` — computed/virtual fields:**

```python
class ProductSerializer(serializers.ModelSerializer):
    is_on_sale = serializers.SerializerMethodField()
    review_count = serializers.SerializerMethodField()
    price_display = serializers.SerializerMethodField()

    class Meta:
        model = Product
        fields = ["id", "name", "price", "is_on_sale", "review_count", "price_display"]

    def get_is_on_sale(self, obj):
        return obj.discount_percent > 0

    def get_review_count(self, obj):
        # ⚠️ N+1 risk! Use annotation in queryset instead for lists
        return obj.reviews.count()

    def get_price_display(self, obj):
        return f"₹{obj.price:,.2f}"
```

**Interview Gotchas:**

| Gotcha                                        | Explanation                                                                               |
| --------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **`read_only_fields` vs `read_only=True`**    | Both work; `read_only_fields` in Meta is shorthand. Use `extra_kwargs` for more control   |
| **`source` + `write_only`**                   | Use together to accept `category_id` on write but expose nested `category` on read        |
| **Nested create/update**                      | DRF does NOT auto-handle writable nested — you MUST override `create()`/`update()`        |
| **`SerializerMethodField` N+1**               | Calling `obj.related.count()` in a list view = N+1. Fix: annotate in queryset             |
| **`fields = "__all__"` in production**        | Never use in production — exposes all fields including sensitive ones. Always be explicit |
| **Validation vs model clean**                 | DRF does NOT call `model.full_clean()` by default. Add it manually if needed              |
| **`to_representation` / `to_internal_value`** | Override for full control over serialization/deserialization (e.g., custom formats)       |

📌 **TLDR:** "Serializer = Django Form for APIs. `ModelSerializer` auto-generates from model; `Serializer` for non-model data. Validation runs in order: field-level → object-level → unique constraints. Use separate read/write serializers for nested data. Override `create()`/`update()` for writable nested. Use `source` to rename/traverse. Never use `fields = '__all__'` in production. Watch for N+1 in `SerializerMethodField`."

---

### 17. Transactions, `atomic`, and `on_commit`

> **📣 Definition:** _"Wrap multi-step DB operations in `transaction.atomic()` (decorator or context manager) to make them all-or-nothing — any exception triggers rollback. Nested calls use SAVEPOINTs, so inner failures can be caught without aborting the outer transaction. Use `transaction.on_commit(callback)` to defer side effects (Celery tasks, webhooks, emails) until the transaction actually commits — without this, a rollback leaves async workers chasing data that doesn't exist. `select_for_update()` for pessimistic row locks (must be inside atomic). Section 11.1 has the deep dive."_

```python
from django.db import transaction

# 1. As a decorator — entire function in one transaction
@transaction.atomic
def transfer_funds(from_account, to_account, amount):
    from_account.balance -= amount
    from_account.save()
    to_account.balance += amount
    to_account.save()
    # If anything raises, BOTH saves are rolled back

# 2. As a context manager — block-level transaction
def complex_op():
    user = User.objects.create(...)
    with transaction.atomic():
        Order.objects.create(user=user, ...)
        Payment.objects.create(...)
        # Rolled back if exception, but user is still created (outside the atomic block)

# 3. Nested atomic — uses SAVEPOINTs
@transaction.atomic
def outer():
    create_user()
    try:
        with transaction.atomic():    # SAVEPOINT
            risky_operation()
    except SomeError:
        # outer transaction continues; inner rolled back to savepoint
        pass

# 4. on_commit — defer side effects until txn commits
@transaction.atomic
def create_order(data):
    order = Order.objects.create(**data)
    # Email is sent ONLY if the transaction commits
    transaction.on_commit(lambda: send_email_task.delay(order.id))
    # If anything below raises, the email task is NEVER queued
    audit_log_create(order)
    return order
```

**Why `on_commit` matters (huge interview talking point):**

```python
# BAD: Email sent before transaction commits
def create_order(data):
    with transaction.atomic():
        order = Order.objects.create(**data)
        send_welcome_email(order.id)   # ❌ runs immediately
        risky_thing()                   # if this fails → order rolled back, but email already sent!

# GOOD: Email sent ONLY after commit
def create_order(data):
    with transaction.atomic():
        order = Order.objects.create(**data)
        transaction.on_commit(lambda: send_welcome_email(order.id))
        risky_thing()
```

#### 📌 TLDR

> "Use `@transaction.atomic` (decorator) or `with transaction.atomic():` (context manager) to wrap multi-step DB operations. Nested `atomic` uses SAVEPOINTs. **`transaction.on_commit(callback)`** — defers side effects (Celery tasks, emails, cache invalidation) until the txn commits, so they don't fire if the txn rolls back. This is critical for data consistency."

---

### 18. Django Caching

> **📣 Definition:** _"Django ships with a unified cache framework supporting multiple backends (Redis, Memcached, database, local-memory). Caching can happen at four levels: **per-site** (cache every page via middleware), **per-view** (decorator on specific views), **template fragment** (cache parts of a template), and **low-level** (cache arbitrary Python objects via `cache.get()/set()`). The hard part isn't caching — it's invalidation. Use TTL-based expiry combined with explicit invalidation on writes, and prefer Redis as the backend for cross-worker consistency."_

**Cache backends** (configured in `settings.py`):

```python
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.redis.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "TIMEOUT": 300,   # 5 min default
    }
}
```

**Levels of caching:**

```python
# 1. Per-view caching — cache entire view response
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)   # 15 min
def product_list(request):
    ...

# 2. Per-fragment caching (in templates)
# {% load cache %}
# {% cache 600 sidebar request.user.id %}
#     ... expensive sidebar HTML ...
# {% endcache %}

# 3. Low-level cache API
from django.core.cache import cache

# Set/get
cache.set("user:42:profile", user_profile, timeout=600)
profile = cache.get("user:42:profile")

# Get-or-set (atomic)
profile = cache.get_or_set("user:42:profile", lambda: fetch_profile(42), timeout=600)

# Many at once
cache.set_many({"k1": "v1", "k2": "v2"})
cache.get_many(["k1", "k2"])

# Delete
cache.delete("user:42:profile")
cache.delete_pattern("user:42:*")   # if backend supports

# 4. Per-site caching (middleware) — caches every page
MIDDLEWARE = [
    "django.middleware.cache.UpdateCacheMiddleware",   # outermost
    # ... others ...
    "django.middleware.cache.FetchFromCacheMiddleware",  # innermost
]
```

**Cache invalidation strategies:**

- **TTL (time-to-live)**: simplest, but stale data possible
- **Versioned keys**: `cache.set(f"user:{user_id}:v{version}", ...)` — increment version on update
- **Signal-based invalidation**: `post_save` clears related cache keys
- **Tag-based invalidation** (via `django-cachalot` or similar)

#### 📌 TLDR

> "Django supports per-view (`@cache_page`), per-fragment (template tag), low-level (`cache.set/get`), and per-site (middleware) caching. Use Redis as the backend. **Cache invalidation is the hard part** — combine TTLs with event-based invalidation via `post_save` signals, and prefer versioned keys over `delete_pattern`."

---

### 19. Django Async — Async Views, ORM, and Concurrency

> **📣 Definition:** _"Django gained async support progressively from 3.1 (async views) through 5.x (mature async ORM). You can write `async def` views to handle concurrent I/O on a single worker, use `async`-suffixed ORM methods (`afilter`, `acreate`, `aget`), and run on an ASGI server (Daphne, Uvicorn) instead of WSGI. Sync code inside async views runs in a threadpool via `sync_to_async`; async code in sync contexts uses `async_to_sync`. Django's async is bolted on, not native — for greenfield async-heavy services, FastAPI is often a better fit."_

Django 4.1+ has progressively added async support; Django 5.x has mature async ORM.

```python
# 1. Async views (Django 3.1+)
async def my_async_view(request):
    async with httpx.AsyncClient() as client:
        resp1, resp2 = await asyncio.gather(
            client.get("https://api1.com"),
            client.get("https://api2.com"),
        )
    return JsonResponse({"data": [resp1.json(), resp2.json()]})


# 2. Async ORM (Django 4.1+)
async def list_products(request):
    # Async iteration
    async for product in Product.objects.filter(status="active"):
        process(product)

    # Async methods (note the 'a' prefix)
    count = await Product.objects.acount()
    product = await Product.objects.aget(id=1)
    products = [p async for p in Product.objects.all()]
    await Product.objects.acreate(name="New")
    await product.asave()
    await product.adelete()


# 3. Mixing sync/async — use sync_to_async / async_to_sync
from asgiref.sync import sync_to_async, async_to_sync

# Call sync code from async context
async def my_view(request):
    result = await sync_to_async(some_sync_function)(arg1, arg2)
    # Useful for legacy code or third-party libs without async support

# Call async code from sync context
def sync_view(request):
    result = async_to_sync(my_async_func)()
```

**Run async Django:**

```bash
# ASGI server (Daphne, Uvicorn, Hypercorn)
uvicorn myproject.asgi:application --workers 4
# Each worker = process with 1 event loop = thousands of concurrent async requests
```

**When to use async in Django:**

- ✅ External API calls in views (use `httpx` not `requests`)
- ✅ Long-polling, Server-Sent Events, WebSockets (Channels)
- ❌ Pure CPU-bound work (use Celery)
- ❌ Talking to a sync-only library (wrap with `sync_to_async` if needed)

#### 📌 TLDR

> "Django 4.1+ has full async support: async views (`async def`), async ORM (`async for`, `await Model.objects.aget()`, `await obj.asave()`). Run with ASGI (Uvicorn/Daphne). Use `sync_to_async`/`async_to_sync` to bridge sync/async code. Best for I/O-bound endpoints (external API calls, real-time features)."

---

### 20. Django Channels — WebSockets & Real-Time

> **📣 Definition:** _"Django Channels extends Django beyond HTTP request/response to handle WebSockets, chat, live notifications, and any long-lived connections. It introduces **Consumers** (analogous to views but for ongoing connections), **Channel Layers** (typically Redis-backed) for cross-process messaging, and **Groups** for broadcasting to multiple connections. Channels turns Django into a fully ASGI app while preserving sync ORM compatibility via `database_sync_to_async`. Production-tested for chat, dashboards, collaborative editing."_

Django Channels extends Django to handle WebSockets, chat, live notifications.

```python
# consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        # Join group
        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.room_group_name, self.channel_name)

    async def receive(self, text_data):
        data = json.loads(text_data)
        message = data["message"]

        # Send message to room group
        await self.channel_layer.group_send(
            self.room_group_name,
            {"type": "chat.message", "message": message},
        )

    async def chat_message(self, event):
        # Handler for "chat.message" type
        await self.send(text_data=json.dumps({"message": event["message"]}))


# routing.py
from django.urls import re_path

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", ChatConsumer.as_asgi()),
]


# asgi.py
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack

application = ProtocolTypeRouter({
    "http": django_asgi_app,
    "websocket": AuthMiddlewareStack(URLRouter(websocket_urlpatterns)),
})
```

**Channel layers** (Redis-backed pub/sub for cross-process communication):

```python
# settings.py
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {"hosts": [("redis", 6379)]},
    },
}
```

#### 📌 TLDR

> "Django Channels extends Django to ASGI for WebSockets, chat, live notifications. Define `AsyncWebsocketConsumer` classes with `connect`/`receive`/`disconnect`. Use **channel layers** (Redis) for cross-worker pub/sub. Run with Daphne/Uvicorn."

---

### 21. Celery — Background Tasks & Scheduled Jobs

> **📣 Definition:** _"Celery is a distributed task queue used with Django for background processing — sending emails, generating reports, processing uploads, retries with backoff, and scheduled jobs (via Celery Beat). Tasks are sent to a broker (Redis or RabbitMQ), workers consume them in separate processes, results optionally stored in a result backend. Critical patterns: idempotent task design, `bind=True` with retry logic, ETA/countdown for delayed execution, task routing to specialized queues, and combining with the Outbox pattern to ensure tasks are only enqueued after the DB transaction commits."_

```python
# celery.py (project root)
from celery import Celery
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")
app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()

# settings.py
CELERY_BROKER_URL = "redis://localhost:6379/0"
CELERY_RESULT_BACKEND = "redis://localhost:6379/1"
CELERY_TASK_SERIALIZER = "json"


# tasks.py (in any app)
from celery import shared_task

@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_email_task(self, user_id, subject, body):
    try:
        user = User.objects.get(id=user_id)
        send_email(user.email, subject, body)
    except SMTPException as exc:
        raise self.retry(exc=exc)   # retry with exponential backoff via 'autoretry_for'

# Call from views
def register_user(request):
    user = User.objects.create(...)
    send_email_task.delay(user.id, "Welcome!", "...")  # queued, non-blocking
    return JsonResponse({"status": "ok"})
```

**Task chaining & groups:**

```python
from celery import chain, group, chord

# Sequential pipeline
chain(task1.s(arg), task2.s(), task3.s()).apply_async()

# Parallel — wait for all
group(task1.s(1), task1.s(2), task1.s(3)).apply_async()

# Parallel + callback
chord(group(fetch.s(url) for url in urls))(merge_results.s())
```

**Periodic tasks (Celery Beat):**

```python
# settings.py
CELERY_BEAT_SCHEDULE = {
    "cleanup-expired-sessions": {
        "task": "myapp.tasks.cleanup_sessions",
        "schedule": crontab(hour=3, minute=0),   # 3am daily
    },
    "send-daily-report": {
        "task": "myapp.tasks.daily_report",
        "schedule": crontab(hour=9, minute=0, day_of_week="monday"),
    },
}
```

**Production-critical: At-least-once task guarantees:**

```python
# For tasks where loss is unacceptable (payments, emails), use:
@shared_task(
    bind=True,
    acks_late=True,              # ACK only AFTER task completes (not when received)
    reject_on_worker_lost=True,  # re-queue if worker crashes mid-task
    max_retries=3,
)
def critical_payment_task(self, payment_id):
    try:
        process_payment(payment_id)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)

# Without acks_late: worker ACKs on receive → worker crashes → task LOST forever.
# With acks_late: task stays on the queue until worker reports success.
# reject_on_worker_lost: if the worker process dies (OOM, segfault),
#   the task is rejected and requeued instead of being lost.
# ⚠️ CRITICAL: tasks MUST be idempotent when using acks_late,
#   because the same task may be delivered more than once.
```

**Testing Celery tasks:**

```python
# settings.py (test config)
CELERY_TASK_ALWAYS_EAGER = True   # tasks run synchronously in tests
CELERY_TASK_EAGER_PROPAGATES = True
```

#### 📌 TLDR

> "Celery for background tasks: define `@shared_task`, call `.delay()` to queue. Use Redis/RabbitMQ as broker. **Always combine with `transaction.on_commit`** so tasks aren't queued if the DB transaction rolls back. Celery Beat for scheduled jobs. Test with `CELERY_TASK_ALWAYS_EAGER=True`."

---

### 22. Migrations — Production-Safe Strategies

> **📣 Definition:** _"Django migrations are auto-generated Python files that track and apply schema changes — `makemigrations` generates them from model diffs, `migrate` applies them. In production, every schema change must be **backward-compatible during the deploy window** because old code may still be running: add columns nullable first, never rename columns directly (add new + dual-write + backfill + drop old), break large data migrations into chunked tasks, and avoid `RunPython` in the same migration as schema changes. Test migrations with `--plan` and on a copy of production before applying."_

```bash
python manage.py makemigrations          # generates migration files from model changes
python manage.py migrate                 # applies pending migrations
python manage.py showmigrations          # shows applied/unapplied
python manage.py sqlmigrate app 0001     # show SQL for a migration
python manage.py migrate app 0005        # migrate to specific version (forward or back)
python manage.py squashmigrations app 0001 0050   # combine many migrations into one
```

**Production-safe migration patterns (tech-lead level):**

1. **Adding a non-nullable field** — break into 3 deploys:

   ```python
   # Deploy 1: Add field as nullable with default
   class Migration(migrations.Migration):
       operations = [
           migrations.AddField(
               model_name="user",
               name="phone",
               field=models.CharField(max_length=20, null=True, blank=True),
           ),
       ]

   # Deploy 2: Backfill data via data migration
   class Migration(migrations.Migration):
       def populate_phones(apps, schema_editor):
           User = apps.get_model("myapp", "User")
           for user in User.objects.filter(phone__isnull=True):
               user.phone = "0000000000"
               user.save()

       operations = [migrations.RunPython(populate_phones, reverse_code=migrations.RunPython.noop)]

   # Deploy 3: Make non-nullable
   class Migration(migrations.Migration):
       operations = [
           migrations.AlterField(
               model_name="user",
               name="phone",
               field=models.CharField(max_length=20),
           ),
       ]
   ```

2. **Renaming a field** — use `RenameField` operation; Django generates it via `makemigrations`.

3. **Dropping a column** — break into 2 deploys:

   - Deploy 1: Stop using the field in code (but don't remove from model)
   - Deploy 2: Remove field from model, generate migration

4. **Long-running migrations** — never block prod:

   - Create indexes with `CONCURRENTLY` (PostgreSQL):
     ```python
     from django.contrib.postgres.operations import AddIndexConcurrently
     class Migration(migrations.Migration):
         atomic = False  # required!
         operations = [
             AddIndexConcurrently(
                 model_name="product",
                 index=models.Index(fields=["name"], name="product_name_idx"),
             )
         ]
     ```

5. **Tools to use:** `django-test-migrations` for testing migration paths, `python manage.py check --deploy` before prod deploys.

#### 📌 TLDR

> "Migrations are version-controlled schema changes. **Production-safe pattern for adding NOT NULL field**: 3 deploys (nullable → backfill → non-null). Use `CONCURRENTLY` for index creation on PostgreSQL (set `atomic = False`). Squash old migrations periodically. Test migration paths with `django-test-migrations` to catch issues before prod."

---

### 23. Custom User Model & Authentication

> **📣 Definition:** _"Django ships with a default User model, but you should **always create a custom user model from day 1** — migrating later is extremely painful and risky. Subclass `AbstractBaseUser` for full control, or `AbstractUser` to keep the default fields and add your own. Common customizations: email instead of username as login, additional profile fields, and custom UserManager. Set `AUTH_USER_MODEL = 'myapp.User'` in settings. This is the single most-cited 'wish I'd done it earlier' decision in Django projects."_

**ALWAYS create a custom user model from day 1** — even if you don't need it now. Migrating later is painful.

```python
# models.py
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models

class UserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra):
        if not email:
            raise ValueError("Email required")
        user = self.model(email=self.normalize_email(email), **extra)
        user.set_password(password)
        user.save()
        return user

    def create_superuser(self, email, password, **extra):
        extra.setdefault("is_staff", True)
        extra.setdefault("is_superuser", True)
        return self.create_user(email, password, **extra)


class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=100)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)
    date_joined = models.DateTimeField(auto_now_add=True)

    objects = UserManager()

    USERNAME_FIELD = "email"   # use email instead of username
    REQUIRED_FIELDS = ["name"]

# settings.py
AUTH_USER_MODEL = "myapp.User"
```

**Authentication backends (custom):**

```python
class EmailBackend:
    def authenticate(self, request, username=None, password=None, **kwargs):
        try:
            user = User.objects.get(email=username)
            if user.check_password(password):
                return user
        except User.DoesNotExist:
            return None

    def get_user(self, user_id):
        try:
            return User.objects.get(pk=user_id)
        except User.DoesNotExist:
            return None

# settings.py
AUTHENTICATION_BACKENDS = ["myapp.backends.EmailBackend"]
```

#### 📌 TLDR

> "Always define a custom user model from project start (`AbstractBaseUser` + `PermissionsMixin` + custom `UserManager`). Set `AUTH_USER_MODEL = 'myapp.User'`. For SSO/SAML/OAuth, write custom authentication backends and add to `AUTHENTICATION_BACKENDS`."

---

### 24. Multi-Database Support & Database Routers

> **📣 Definition:** _"Django supports multiple databases simultaneously via the `DATABASES` setting and Database Routers — Python classes that decide which DB to use for each query (`db_for_read`, `db_for_write`, `allow_relation`, `allow_migrate`). Common patterns: read replicas (route reads to replica, writes to primary), separate analytics DB for reporting queries, sharding by tenant. Use `Model.objects.using('replica').filter(...)` for explicit DB selection. Replication lag means read-after-write needs careful handling — sometimes route critical reads to primary."_

```python
# settings.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "main",
        # ...
    },
    "analytics": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "analytics",
        # ...
    },
}

DATABASE_ROUTERS = ["myapp.routers.AnalyticsRouter"]


# routers.py
class AnalyticsRouter:
    """Route analytics-related models to the analytics DB."""

    analytics_models = {"pageview", "userevent", "abtestresult"}

    def db_for_read(self, model, **hints):
        if model._meta.model_name in self.analytics_models:
            return "analytics"
        return "default"

    def db_for_write(self, model, **hints):
        if model._meta.model_name in self.analytics_models:
            return "analytics"
        return "default"

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        if model_name in self.analytics_models:
            return db == "analytics"
        return db == "default"


# Manual selection in code
User.objects.using("replica").all()       # read from replica
user.save(using="default")                 # write to primary
```

**Read replicas pattern:**

```python
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return random.choice(["replica1", "replica2"])

    def db_for_write(self, model, **hints):
        return "primary"
```

#### 📌 TLDR

> "Configure multiple DBs in `DATABASES` and write a router (`db_for_read`, `db_for_write`, `allow_migrate`) to direct queries. Common use: read replicas (write to primary, read from replicas) or separating analytics from operational data. Use `Model.objects.using('db_alias')` for explicit routing."

---

### 25. Testing in Django

> **📣 Definition:** _"Django's testing framework wraps each test in a transaction that's rolled back automatically — fast and isolated. Use `TestCase` for normal tests (transaction rollback), `TransactionTestCase` when you need real commits or to test atomic blocks themselves, and `setUpTestData` to share read-only fixtures across a test class for speed. Use `Client` for view tests, `APIClient` for DRF, factory-boy or model-bakery for test data generation, and pytest-django for cleaner fixture syntax. Always test against a real Postgres in Docker — SQLite has different behaviors that mask bugs."_

```python
from django.test import TestCase, TransactionTestCase, Client
from django.urls import reverse
from rest_framework.test import APIClient

# Basic model test
class ProductModelTest(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Runs ONCE for the entire test class — much faster than setUp
        cls.category = Category.objects.create(name="Electronics")

    def setUp(self):
        # Runs before EACH test
        self.product = Product.objects.create(name="Phone", category=self.category, price=10000)

    def test_str_representation(self):
        self.assertEqual(str(self.product), "Phone")

    def test_query_count(self):
        # Prevent N+1 regressions
        with self.assertNumQueries(1):
            list(Product.objects.select_related("category").all())


# View test with Client
class ProductViewTest(TestCase):
    def test_product_list_loads(self):
        response = self.client.get(reverse("product-list"))
        self.assertEqual(response.status_code, 200)
        self.assertContains(response, "Products")


# DRF test
class ProductAPITest(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(email="t@t.com", password="x")
        self.client.force_authenticate(user=self.user)

    def test_create_product(self):
        response = self.client.post("/api/products/", {"name": "Test", "price": 100}, format="json")
        self.assertEqual(response.status_code, 201)


# Mocking external APIs
from unittest.mock import patch

class ExternalAPITest(TestCase):
    @patch("myapp.services.requests.get")
    def test_payment(self, mock_get):
        mock_get.return_value.json.return_value = {"status": "ok"}
        result = process_payment(...)
        self.assertTrue(result.success)
```

**`TestCase` vs `TransactionTestCase`:**

- **`TestCase`**: Wraps each test in a transaction that gets rolled back. **Fast.** But signals like `transaction.on_commit` won't fire (because nothing actually commits).
- **`TransactionTestCase`**: Truncates tables between tests. **Slower.** Use when you need actual transaction commits (testing on_commit hooks, multi-DB).

#### 📌 TLDR

> "Use `TestCase` (transaction-rollback, fast) by default. Use `TransactionTestCase` only when testing `on_commit` hooks. Use `setUpTestData` (runs once per class) over `setUp` (runs per test) for shared fixtures. Use `assertNumQueries` to catch N+1 regressions. Mock external APIs with `unittest.mock.patch`. For DRF, use `APIClient` and `force_authenticate`."

---

### 26. Django Security — What Tech Leads Must Know

> **📣 Definition:** _"Django ships with strong defaults against the OWASP Top 10: ORM parameterization (SQL injection), template auto-escaping (XSS), `CsrfViewMiddleware` (CSRF), `XFrameOptionsMiddleware` (clickjacking), `ALLOWED_HOSTS` (host header attacks), PBKDF2 password hashing, and secure session cookies. The tech-lead's job is to know which defaults are active, configure HTTPS-only cookies in production (`SESSION_COOKIE_SECURE`), set strict `Content-Security-Policy` headers, audit any use of `\|safe`/`mark_safe`/`raw()`, and treat security middleware order as deliberate, not accidental."_

| Vulnerability           | Django Protection                        | Your Job                                                            |
| ----------------------- | ---------------------------------------- | ------------------------------------------------------------------- |
| **SQL Injection**       | ORM parameterizes by default             | Don't use `.raw()` with user input; if you must, use `params=[]`    |
| **XSS**                 | Templates auto-escape by default         | Don't use `\|safe` filter or `mark_safe()` on user input            |
| **CSRF**                | `CsrfViewMiddleware`                     | Include `{% csrf_token %}` in forms; `csrfmiddlewaretoken` for AJAX |
| **Clickjacking**        | `XFrameOptionsMiddleware`                | Set `X_FRAME_OPTIONS = "DENY"`                                      |
| **Host header attacks** | `ALLOWED_HOSTS` setting                  | Always set explicitly in production                                 |
| **Session hijacking**   | Secure session cookies                   | `SESSION_COOKIE_SECURE = True`, `SESSION_COOKIE_HTTPONLY = True`    |
| **Password storage**    | PBKDF2 by default                        | Consider Argon2 (`PASSWORD_HASHERS`) for higher security            |
| **Mass assignment**     | Forms/serializers validate               | Use `fields = [...]`, never `fields = "__all__"` for public APIs    |
| **Open redirects**      | `url_has_allowed_host_and_scheme` helper | Validate `next` URL params                                          |

**Production checklist:**

```bash
python manage.py check --deploy
# Reports issues like missing SECURE_HSTS_SECONDS, DEBUG=True, etc.
```

```python
# Production settings essentials
DEBUG = False
ALLOWED_HOSTS = ["myapp.com", "www.myapp.com"]
SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000   # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
X_FRAME_OPTIONS = "DENY"
```

#### 📌 TLDR

> "Django auto-protects against SQL injection (ORM), XSS (template escaping), and CSRF (middleware). Tech-lead duties: set `ALLOWED_HOSTS`, secure cookie flags, HSTS, never `fields='__all__'` for public APIs, run `manage.py check --deploy` before every prod release."

---

### 27. Performance & Scaling — Tech-Lead Talking Points

> **📣 Definition:** _"Scaling Django at the tech-lead level is a stack of disciplines: ORM optimization (kill N+1 with `select_related`/`prefetch_related`, project columns with `.only()`/`.defer()`, paginate everything), caching tiers (Redis for sessions and computed data, CDN for static), database tuning (indexes guided by `EXPLAIN ANALYZE`, read replicas, PgBouncer for connection pooling), async views and Celery for I/O-bound and slow work, and observability (`django-silk`, OpenTelemetry, Sentry) to know what to optimize. The wrong answer is 'we'll add caching' — the right one is 'measure first, then optimize the proven bottleneck'."_

**Optimization checklist:**

1. **Query optimization** — `select_related`/`prefetch_related`, `assertNumQueries` in tests, Django Debug Toolbar in dev.
2. **Caching layers** — Redis for sessions, querysets, computed views; CDN for static assets.
3. **Database** — proper indexes, `EXPLAIN ANALYZE`, read replicas for heavy reads, connection pooling (`pgbouncer`/`django-db-connection-pool`).
4. **Async views** for I/O-bound endpoints (external API calls).
5. **Celery** for slow operations (emails, image processing, reports).
6. **Pagination** — never return unbounded querysets.
7. **`.only()` / `.defer()`** — fetch only needed fields:
   ```python
   Product.objects.only("id", "name", "price")   # only these columns
   Product.objects.defer("description")          # everything except this
   ```
8. **Bulk operations** — `bulk_create`, `bulk_update`, `update()` instead of loop+save.
9. **`iterator()` for huge result sets** — streams from DB instead of loading all into memory.
10. **Connection pooling** — set `CONN_MAX_AGE` in DB config.

**Deployment stack (production):**

```
[Cloudflare/CDN]
       │
       ▼
[Nginx (TLS, static files)]
       │
       ▼
[Gunicorn (WSGI) or Uvicorn (ASGI)]   ← multiple workers
       │
       ▼
[Django app (multiple processes)]
       │            │              │
       ▼            ▼              ▼
[PostgreSQL] [Redis cache]    [Celery workers]
[+ replicas]                   [+ Celery Beat]
```

#### 📌 TLDR

> "Performance: kill N+1, cache aggressively (Redis), use read replicas, paginate everything, use `bulk_*` for batch ops, `iterator()` for huge result sets, async for I/O-bound, Celery for slow tasks. Production stack: Nginx → Gunicorn/Uvicorn workers → Django → Postgres + Redis + Celery."

---

### 28. Common Tech-Lead Interview Questions & How to Answer Them

> **📣 Definition:** _"This is a curated set of the questions that come up most often in Django tech-lead interviews — request lifecycle, fixing N+1, when NOT to use signals, handling long-running tasks, scaling to 10k RPS, project structure at scale, and production-safe schema changes. The pattern in good answers: name the concept, give a concrete example from a system you've built, and acknowledge the trade-off. Generic answers signal junior; trade-off-aware answers signal senior."_

| Question                                                      | Key points to cover                                                                                                                                                  |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Walk me through the Django request lifecycle."               | WSGI/ASGI → middleware → URL resolver → view → response → middleware reverse                                                                                         |
| "How do you find and fix N+1 queries?"                        | Django Debug Toolbar / SQL logging → `select_related`/`prefetch_related` → `assertNumQueries` to prevent regressions                                                 |
| "When would you NOT use signals?"                             | When the calling code can call the function explicitly; service-layer pattern is cleaner; signals don't fire on `bulk_create`                                        |
| "How do you handle a long-running task in a Django view?"     | Don't block the view — use Celery, return 202 Accepted with a task ID, client polls for status                                                                       |
| "How do you structure a Django project at scale?"             | Apps as bounded contexts, service-layer pattern, custom managers/QuerySets, separate `settings/` directory (base, dev, prod)                                         |
| "How do you handle DB schema changes in production?"          | Multi-deploy migrations (nullable → backfill → enforce), `CONCURRENTLY` indexes, test with `django-test-migrations`                                                  |
| "How do you scale a Django app to 10k req/s?"                 | Read replicas, async views for I/O, Redis caching at multiple layers, CDN, horizontal scaling with multiple Gunicorn/Uvicorn workers, Celery for slow tasks          |
| "Custom user model — when and why?"                           | Always from day 1 (`AbstractBaseUser` + `PermissionsMixin`); migrating later is painful                                                                              |
| "Explain Django's transaction model."                         | `@atomic` decorator/context manager, nested via SAVEPOINTs, `on_commit` for side effects post-commit                                                                 |
| "Difference between `select_related` and `prefetch_related`?" | JOIN vs separate query+Python join; FK/O2O vs M2M/reverse-FK; combine via chaining                                                                                   |
| "How do you implement RBAC?"                                  | Django's `Group` + `Permission` for simple cases; custom permission classes / object-level perms (`django-guardian`) for complex; encode in DRF `permission_classes` |
| "How do you optimize Django ORM?"                             | `select_related`/`prefetch_related`, `only`/`defer`, `bulk_*`, indexed fields, `assertNumQueries`, custom managers, `EXPLAIN`                                        |

#### 📌 TLDR

> "Tech-lead Django interviews focus on: production-grade ORM optimization, scaling strategies, migration safety, signal pitfalls, custom user models from day 1, async + Celery for I/O and background tasks, and architectural decisions (CBV vs FBV, signals vs services, monolith vs splitting apps). Lead with concrete examples from real systems."

---

### 29. API Versioning in Django / DRF

> **📣 Definition:** _"API versioning lets you evolve your API without breaking existing clients. DRF supports 5 strategies out of the box: URL path (`/api/v1/`), query parameter (`?version=1`), header (`Accept: application/vnd.myapp.v2+json`), namespace, and hostname versioning. The tech-lead decision: URL-path versioning is the most common and debuggable; header-based is the most RESTful. Internally, use separate serializers per version, not if-else spaghetti in views."_

```python
# === settings.py — enable versioning ===
REST_FRAMEWORK = {
    "DEFAULT_VERSIONING_CLASS": "rest_framework.versioning.URLPathVersioning",
    "DEFAULT_VERSION": "v1",
    "ALLOWED_VERSIONS": ["v1", "v2"],
    "VERSION_PARAM": "version",
}

# === urls.py — URL path versioning (most common) ===
urlpatterns = [
    path("api/<version>/", include("myapp.urls")),
    # /api/v1/products/, /api/v2/products/
]

# === views.py — version-aware ViewSet ===
class ProductViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == "v2":
            return ProductSerializerV2       # new fields, different structure
        return ProductSerializerV1

    def get_queryset(self):
        if self.request.version == "v2":
            return Product.objects.select_related("category", "brand")
        return Product.objects.select_related("category")


# === Serializer per version (clean separation) ===
class ProductSerializerV1(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = ["id", "name", "price"]

class ProductSerializerV2(serializers.ModelSerializer):
    category = CategorySerializer(read_only=True)
    brand = BrandSerializer(read_only=True)

    class Meta:
        model = Product
        fields = ["id", "name", "price", "sku", "category", "brand", "created_at"]
```

**Versioning strategies comparison:**

| Strategy          | URL looks like                          | Pros                           | Cons                            |
| ----------------- | --------------------------------------- | ------------------------------ | ------------------------------- |
| **URL Path**      | `/api/v1/users/`                        | Visible, debuggable, cacheable | URL changes between versions    |
| **Query Param**   | `/api/users/?version=1`                 | Easy to add                    | Easy to forget, cache issues    |
| **Accept Header** | `Accept: application/vnd.myapp.v2+json` | Most RESTful, clean URLs       | Hidden, hard to test in browser |
| **Namespace**     | Different URL conf per version          | Full isolation                 | Code duplication                |

📌 **TLDR:** "Use URL-path versioning (`/api/v1/`) — most common, most debuggable. Separate serializers per version, not `if-else` in fields. Set `ALLOWED_VERSIONS` to reject unknown versions. Deprecate old versions with `Sunset` header + docs timeline."

---

### 30. JWT Authentication & RBAC in Django

> **📣 Definition:** _"JWT (JSON Web Token) handles **authentication** (proving who you are) — not authorization. The token is a signed, stateless credential containing user identity and expiry. **Authorization** (what you can access) is handled by Django's permissions system — Groups, Permissions, and custom DRF permission classes. RBAC (Role-Based Access Control) maps: User → Groups (roles) → Permissions (capabilities). JWT proves identity; Django's permission layer decides access."_

```python
# === JWT Setup (djangorestframework-simplejwt) ===
# pip install djangorestframework-simplejwt

# settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": [
        "rest_framework_simplejwt.authentication.JWTAuthentication",
    ],
}

SIMPLE_JWT = {
    "ACCESS_TOKEN_LIFETIME": timedelta(minutes=15),    # short-lived
    "REFRESH_TOKEN_LIFETIME": timedelta(days=7),       # longer
    "ROTATE_REFRESH_TOKENS": True,                     # new refresh on each use
    "BLACKLIST_AFTER_ROTATION": True,                  # invalidate old refresh token
    "ALGORITHM": "HS256",
    "SIGNING_KEY": SECRET_KEY,
    "AUTH_HEADER_TYPES": ("Bearer",),
}

# urls.py
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
    path("api/token/", TokenObtainPairView.as_view()),       # POST email+password → access+refresh
    path("api/token/refresh/", TokenRefreshView.as_view()),  # POST refresh → new access
]


# === Authentication vs Authorization Flow ===
# 1. AUTHENTICATION (JWT):
#    Client sends: Authorization: Bearer <access_token>
#    DRF's JWTAuthentication middleware:
#      - Decodes token, verifies signature + expiry
#      - Sets request.user = User object from token's user_id
#      - If expired → 401 Unauthorized
#
# 2. AUTHORIZATION (Django Permissions / RBAC):
#    After authentication, DRF checks permission_classes:
#      - Is user in the right Group (role)?
#      - Does user have the required Permission?
#      - Does user have object-level access?


# === RBAC with Django Groups & Permissions ===
# Setup in admin or migration:
from django.contrib.auth.models import Group, Permission

# Create roles
admin_group = Group.objects.create(name="admin")
editor_group = Group.objects.create(name="editor")
viewer_group = Group.objects.create(name="viewer")

# Assign permissions to roles
from django.contrib.contenttypes.models import ContentType
ct = ContentType.objects.get_for_model(Product)

can_edit = Permission.objects.get(codename="change_product", content_type=ct)
can_delete = Permission.objects.get(codename="delete_product", content_type=ct)

editor_group.permissions.add(can_edit)
admin_group.permissions.set(Permission.objects.filter(content_type=ct))  # all product perms

# Assign user to role
user.groups.add(editor_group)


# === Custom Permission Class (DRF) ===
from rest_framework.permissions import BasePermission

class IsAdminOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        if request.method in ("GET", "HEAD", "OPTIONS"):
            return True
        return request.user.groups.filter(name="admin").exists()

class IsOwnerOrAdmin(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user or request.user.is_staff


# === Custom JWT Claims (add role/permissions to token) ===
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token["email"] = user.email
        token["role"] = list(user.groups.values_list("name", flat=True))
        token["permissions"] = list(user.get_all_permissions())
        return token
```

**JWT + RBAC flow diagram:**

```
Client                          Server
  │                               │
  ├── POST /api/token/ ──────────→│  (Authentication)
  │   {email, password}           │   verify credentials
  │                               │   generate JWT with user_id + role
  │←── {access, refresh} ────────│
  │                               │
  ├── GET /api/products/ ────────→│  (Authorization)
  │   Authorization: Bearer xxx   │   1. Decode JWT → request.user
  │                               │   2. Check permission_classes
  │                               │   3. Check object permissions
  │←── 200 / 403 ────────────────│
```

📌 **TLDR:** "JWT handles authentication (who you are) — it's a signed, stateless token with user_id and expiry. Authorization (what you can do) is handled by Django's Groups + Permissions = RBAC. Use `djangorestframework-simplejwt`. Short-lived access tokens (15 min), longer refresh tokens (7 days). Add role/permissions to JWT claims for frontend use. Custom `BasePermission` classes for fine-grained access control."

---

### 31. Django Security — Extended Deep Dive

> **📣 Definition:** _"Beyond the OWASP basics covered in 15.24, tech leads are expected to discuss: rate limiting (django-ratelimit or DRF throttling), Content-Security-Policy headers, CORS configuration, secrets management, dependency auditing, and how Django's middleware ordering affects security. The key insight: Django's defaults are strong but not sufficient — you must actively configure production hardening."_

```python
# === CORS (Cross-Origin Resource Sharing) ===
# pip install django-cors-headers

INSTALLED_APPS = [..., "corsheaders"]
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",     # MUST be before CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    ...
]

# Production: whitelist specific origins
CORS_ALLOWED_ORIGINS = [
    "https://myapp.com",
    "https://admin.myapp.com",
]
CORS_ALLOW_CREDENTIALS = True                    # for cookies/auth headers


# === Content-Security-Policy ===
# pip install django-csp
MIDDLEWARE += ["csp.middleware.CSPMiddleware"]

CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC = ("'self'", "https://cdn.jsdelivr.net")
CSP_STYLE_SRC = ("'self'", "'unsafe-inline'")     # needed for some CSS frameworks
CSP_IMG_SRC = ("'self'", "data:", "https://images.myapp.com")


# === Rate Limiting ===
# Option 1: DRF throttling (API-level)
REST_FRAMEWORK = {
    "DEFAULT_THROTTLE_CLASSES": [
        "rest_framework.throttling.ScopedRateThrottle",
    ],
    "DEFAULT_THROTTLE_RATES": {
        "login": "5/min",        # prevent brute-force
        "api": "100/min",
        "sensitive": "10/min",
    },
}

# Option 2: django-ratelimit (view-level)
from django_ratelimit.decorators import ratelimit

@ratelimit(key="ip", rate="5/m", method="POST", block=True)
def login_view(request):
    ...


# === Secrets Management ===
# NEVER hardcode secrets. Use environment variables + django-environ:
import environ
env = environ.Env()
SECRET_KEY = env("DJANGO_SECRET_KEY")
DATABASE_URL = env("DATABASE_URL")
# In production: AWS Secrets Manager, HashiCorp Vault, or GCP Secret Manager


# === Dependency Auditing ===
# pip install pip-audit safety
# pip-audit                    # scan installed packages for known vulnerabilities
# safety check                 # check against safety DB
# Run in CI/CD pipeline!


# === Security Middleware Order (matters!) ===
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",     # FIRST — HSTS, SSL redirect
    "corsheaders.middleware.CorsMiddleware",              # before CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    "django.contrib.sessions.middleware.SessionMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",         # before auth
    "django.contrib.auth.middleware.AuthenticationMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
    "csp.middleware.CSPMiddleware",                       # after all response-modifying middleware
]
```

📌 **TLDR:** "Beyond Django's built-in protections: configure CORS (whitelist origins), add CSP headers, rate-limit login endpoints (5/min), manage secrets via env vars (never hardcode), audit dependencies with `pip-audit`, and ensure middleware ordering is correct (SecurityMiddleware first, CORS before CommonMiddleware)."

---

### 32. Migration Management — Versioning & Production Workflows

> **📣 Definition:** _"Beyond the basics in 15.20, tech leads manage migrations as a version-controlled deployment artifact. Key concerns: migration conflicts in teams (merge migrations), squashing old migrations for performance, data migration safety, rollback strategies, and zero-downtime deployments. The golden rule: migrations must be backward-compatible with the currently running code during the deploy window."_

```python
# === Handling Migration Conflicts in Teams ===
# Two developers create migrations on the same app → conflict
# Solution: Django auto-detects and generates a merge migration
python manage.py makemigrations --merge
# Creates: 0005_merge_0003_feature_a_0004_feature_b.py

# Prevention: communicate, use feature branches, run makemigrations in CI


# === Squashing Migrations (clean up old history) ===
python manage.py squashmigrations myapp 0001 0050
# Combines 50 migrations into 1 optimized file
# Old migrations kept as dependencies until fully applied everywhere
# After confirming all environments have migrated past 0050:
#   - Delete old 0001-0050 files
#   - Remove replaces list from squashed migration


# === Data Migrations — Safe Patterns ===
from django.db import migrations

def forwards_func(apps, schema_editor):
    """Always use apps.get_model() — NOT direct model imports!"""
    User = apps.get_model("myapp", "User")     # historical model, not current
    # Chunk large updates to avoid locking
    batch_size = 1000
    while True:
        batch = list(User.objects.filter(role__isnull=True)[:batch_size])
        if not batch:
            break
        for user in batch:
            user.role = "member"
        User.objects.bulk_update(batch, ["role"])

def reverse_func(apps, schema_editor):
    User = apps.get_model("myapp", "User")
    User.objects.filter(role="member").update(role=None)

class Migration(migrations.Migration):
    dependencies = [("myapp", "0010_add_role_field")]
    operations = [
        migrations.RunPython(forwards_func, reverse_func),
    ]
# ⚠️ ALWAYS provide reverse_code for rollback!
# ⚠️ NEVER import models directly — use apps.get_model()


# === Rollback Strategy ===
python manage.py migrate myapp 0009          # roll back to migration 0009
# Only works if all migrations have reverse operations defined
# Schema-only migrations auto-generate reverse; RunPython needs manual reverse


# === Zero-Downtime Deploy Workflow ===
# 1. Deploy NEW code that works with BOTH old and new schema
# 2. Run migrations (background, non-blocking)
# 3. Verify, then deploy code that uses new schema only
# 4. Clean up: remove old field handling code
#
# Example: Renaming a column
# Deploy 1: Add new column (nullable), write to both old+new
# Deploy 2: Backfill new column from old
# Deploy 3: Read from new column, stop writing to old
# Deploy 4: Drop old column


# === CI/CD Migration Checks ===
# In CI pipeline:
python manage.py makemigrations --check --dry-run   # fail if unapplied model changes
python manage.py migrate --plan                      # show what will run
python manage.py test                                # migrations auto-applied in test DB
```

📌 **TLDR:** "Migrations are deployment artifacts — version-controlled, backward-compatible, reversible. Use `--merge` for team conflicts, `squashmigrations` to clean history. Data migrations: always use `apps.get_model()`, provide `reverse_code`, chunk large updates. Zero-downtime: deploy code that handles both schemas → migrate → deploy final code. Run `makemigrations --check` in CI."

---

### 33. Horizontally Scaling a Django Application

> **📣 Definition:** _"Horizontal scaling means adding more server instances rather than bigger servers. Django scales horizontally because it's stateless per-request — but you must externalize all state: sessions to Redis, file uploads to S3, cache to Redis/Memcached, background jobs to Celery+broker, and database connections through a pooler. The load balancer distributes requests; each Django instance is identical and disposable."_

**The horizontal scaling architecture:**

```
                    ┌──────────────────┐
                    │   Load Balancer  │
                    │  (Nginx / ALB)   │
                    └────┬───┬───┬─────┘
                         │   │   │
              ┌──────────┘   │   └──────────┐
              ▼              ▼              ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │  Django #1  │ │  Django #2  │ │  Django #3  │    ← identical, stateless
    │  Gunicorn   │ │  Gunicorn   │ │  Gunicorn   │
    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
           │               │               │
    ┌──────┴───────────────┴───────────────┴──────┐
    │              Shared Infrastructure           │
    │                                              │
    │  ┌──────────┐  ┌───────┐  ┌──────────────┐  │
    │  │ Postgres │  │ Redis │  │ Celery       │  │
    │  │ (Primary │  │ cache │  │ Workers (N)  │  │
    │  │ +Replica)│  │+sessions│ │ + Beat       │  │
    │  └──────────┘  └───────┘  └──────────────┘  │
    │  ┌──────────┐  ┌──────────────────────────┐  │
    │  │PgBouncer │  │ S3 / MinIO (file storage)│  │
    │  └──────────┘  └──────────────────────────┘  │
    └──────────────────────────────────────────────┘
```

**The 7 things you must externalize:**

| Component              | Local (single-server) | Scaled (multi-server)                     |
| ---------------------- | --------------------- | ----------------------------------------- |
| **Sessions**           | DB-backed (default)   | Redis (`django-redis` + `SESSION_ENGINE`) |
| **Cache**              | Local memory          | Redis cluster (shared across instances)   |
| **Static/media files** | Local filesystem      | S3 / CloudFront CDN                       |
| **Background tasks**   | In-process            | Celery + Redis/RabbitMQ broker            |
| **DB connections**     | Direct                | PgBouncer (connection pooler)             |
| **WebSockets**         | In-memory             | Redis pub/sub (Django Channels)           |
| **Cron/scheduled**     | System cron           | Celery Beat (single instance)             |

```python
# === Session to Redis ===
# pip install django-redis
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://redis:6379/0",
    }
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
SESSION_CACHE_ALIAS = "default"


# === Static files to S3 ===
# pip install django-storages boto3
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
STATICFILES_STORAGE = "storages.backends.s3boto3.S3StaticStorage"
AWS_STORAGE_BUCKET_NAME = "myapp-assets"
AWS_S3_CUSTOM_DOMAIN = "cdn.myapp.com"


# === Database connection pooling ===
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "HOST": "pgbouncer",           # connect through PgBouncer, not directly
        "PORT": 6432,                  # PgBouncer port
        "CONN_MAX_AGE": 0,            # let PgBouncer manage connection lifetime
    },
    "replica": {                       # read replica for heavy reads
        "ENGINE": "django.db.backends.postgresql",
        "HOST": "pgbouncer-replica",
    },
}

# Database router for read/write splitting
class PrimaryReplicaRouter:
    def db_for_read(self, model, **hints):
        return "replica"
    def db_for_write(self, model, **hints):
        return "default"
    def allow_relation(self, obj1, obj2, **hints):
        return True
    def allow_migrate(self, db, app_label, **hints):
        return db == "default"

DATABASE_ROUTERS = ["myapp.routers.PrimaryReplicaRouter"]


# === Gunicorn config (per instance) ===
# gunicorn myapp.wsgi:application \
#   --workers 4 \                    # 2×CPU+1
#   --threads 2 \                    # for I/O-bound views
#   --bind 0.0.0.0:8000 \
#   --max-requests 1000 \            # restart workers after N requests (memory leak prevention)
#   --max-requests-jitter 50 \
#   --timeout 30 \
#   --access-logfile -
```

**Scaling checklist:**

```
1. ✅ Stateless Django instances (no local state)
2. ✅ Sessions → Redis
3. ✅ Cache → Redis
4. ✅ Files → S3 + CDN
5. ✅ DB → PgBouncer + read replicas
6. ✅ Background tasks → Celery + broker
7. ✅ Load balancer → Nginx / ALB (sticky sessions OFF)
8. ✅ Health check endpoint → /healthz/ (LB probes)
9. ✅ Graceful shutdown → Gunicorn handles SIGTERM
10. ✅ Auto-scaling → container orchestration (K8s / ECS)
```

📌 **TLDR:** "Django scales horizontally because requests are stateless — but you must externalize ALL state: sessions → Redis, files → S3, cache → Redis, tasks → Celery, DB connections → PgBouncer. Each instance is identical and disposable. Load balancer distributes traffic. Use read replicas for heavy reads. Auto-scale with K8s or ECS. `Gunicorn --workers 2×CPU+1`."

---

## 34. FastAPI — Complete Guide for Technical Lead Role

> **📣 Interview-ready definition:** _"FastAPI is a modern, async-first Python web framework for building APIs, built on Starlette (ASGI toolkit) + Pydantic (validation) + Dependency Injection + auto-generated OpenAPI docs. Unlike Django, it's API-focused with no built-in admin or templates, but its async-native design and type-driven validation make it the fastest mainstream Python framework — comparable to Node.js and Go. For tech-lead interviews, the depth lies in the async/sync endpoint discipline (sync I/O inside `async def` blocks the entire event loop), the dependency injection system (used for DB sessions, auth, services), Pydantic v2 patterns (separating input/output models, settings management), middleware vs dependencies, project structure for large apps, and production deployment with Gunicorn + UvicornWorker."_

This section covers FastAPI from foundational to advanced. Even if you've used FastAPI for years, the deeper topics (DI scoping, middleware execution order, lifespan events, async pitfalls, project structure for large apps) are what tech-lead interviews test. Section 7 covers the basic async/FastAPI patterns; this section drills into framework-specific concepts.

---

### 35. ASGI vs WSGI — The Foundation

> **📣 Definition:** _"WSGI (Web Server Gateway Interface) is the synchronous Python web standard used by Django and Flask — one request per thread, no support for WebSockets or long-lived connections. ASGI (Asynchronous Server Gateway Interface) is the modern async successor — it natively supports HTTP, WebSockets, and Server-Sent Events on a single-threaded event loop. FastAPI is built on Starlette (ASGI toolkit) and runs on Uvicorn (ASGI server using uvloop + httptools, both C extensions) for performance comparable to Node.js and Go."_

#### 🧑‍🍳 Layman

WSGI (Django/Flask) is like a single phone line — one call at a time. ASGI (FastAPI) is like a switchboard — many calls handled simultaneously, with support for streaming, WebSockets, and bidirectional communication.

#### 🔧 Technical

|                        | WSGI                         | ASGI                                     |
| ---------------------- | ---------------------------- | ---------------------------------------- |
| Concurrency            | Synchronous, threaded        | Asynchronous, single-threaded event loop |
| Connection types       | HTTP only                    | HTTP, WebSockets, Server-Sent Events     |
| Long-lived connections | ❌ Awkward                   | ✅ Native                                |
| Frameworks             | Django (legacy), Flask       | FastAPI, Starlette, Sanic, Litestar      |
| Servers                | Gunicorn (with sync workers) | Uvicorn, Hypercorn, Daphne               |

**FastAPI's stack:**

```
Your code        ← FastAPI (routing + DI + Pydantic)
                 ↑
                 ↓
Starlette        ← ASGI toolkit (middleware, routing primitives)
                 ↑
                 ↓
Uvicorn          ← ASGI server (uvloop + httptools — C extensions for speed)
                 ↑
                 ↓
OS sockets
```

FastAPI is essentially **Starlette + Pydantic + Dependency Injection + OpenAPI** — that's the entire framework in one sentence.

📌 **TLDR:** "FastAPI is built on Starlette (ASGI toolkit) and Pydantic (validation). ASGI replaces WSGI for async-first apps with WebSockets and SSE support. Uvicorn is the production ASGI server, using uvloop and httptools (C extensions) for performance comparable to Node.js and Go."

---

### 36. Path Operations, Parameters & Routing

> **📣 Definition:** _"FastAPI uses Python type hints to derive parameter sources — the same `int` annotation can mean parsing from path, query, or body depending on declaration. `Path()`, `Query()`, `Body()`, `Header()`, `Cookie()` add validation constraints (length, range, regex). Use `Annotated[type, Query(...)]` (modern style) instead of `= Query(...)` for cleaner type hints. Organize endpoints with `APIRouter` per feature/domain and mount via `app.include_router()`. Path matching is order-sensitive — declare specific routes (`/users/me`) before parameterized ones (`/users/{id}`)."_

```python
from fastapi import FastAPI, Path, Query, Body, Header, Cookie
from typing import Annotated

app = FastAPI(title="My API", version="1.0.0", description="...")

# ── Path parameters ─────────────────────────────────────────
@app.get("/users/{user_id}")
async def get_user(user_id: int):                     # automatic int conversion
    return {"user_id": user_id}

# Validation via Annotated (modern style, Python 3.9+)
@app.get("/users/{user_id}")
async def get_user(
    user_id: Annotated[int, Path(ge=1, le=10**9, description="User ID")]
):
    return {"user_id": user_id}

# ── Query parameters ────────────────────────────────────────
@app.get("/search")
async def search(
    q: Annotated[str, Query(min_length=3, max_length=50)],
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    tags: Annotated[list[str], Query()] = []          # ?tags=a&tags=b
):
    return {"q": q, "limit": limit, "tags": tags}

# ── Request body ────────────────────────────────────────────
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str

@app.post("/users")
async def create_user(user: UserCreate):              # parses + validates JSON body
    return user

# ── Headers and cookies ─────────────────────────────────────
@app.get("/me")
async def me(
    authorization: Annotated[str, Header()],
    session_id: Annotated[str | None, Cookie()] = None,
):
    return {"auth": authorization, "session": session_id}

# ── Routers (organize by feature) ───────────────────────────
from fastapi import APIRouter

users_router = APIRouter(prefix="/users", tags=["users"])

@users_router.get("/")
async def list_users(): ...

@users_router.get("/{user_id}")
async def get_user(user_id: int): ...

orders_router = APIRouter(prefix="/orders", tags=["orders"])

# Mount on app
app.include_router(users_router)
app.include_router(orders_router)
# Now: GET /users, GET /users/{id}, GET /orders, ...
```

**Order matters!** Path operations are matched in the order they're registered:

```python
@app.get("/users/me")           # matches /users/me
async def me(): ...

@app.get("/users/{user_id}")    # matches /users/anything-else
async def get_user(user_id: int): ...
# If reversed, /users/me would try to parse "me" as int → 422 error
```

📌 **TLDR:** "FastAPI uses Python type hints to derive parameter sources — the same `int` annotation means parsing from path, query, or body depending on declaration. Use `Annotated[type, Query/Path/Body(...)]` for validation constraints. Organize endpoints with `APIRouter` per feature/domain. Path matching is order-sensitive — declare specific routes (`/users/me`) before parameterized ones (`/users/{id}`)."

---

### 37. Pydantic Models — Validation, Serialization, Settings

> **📣 Definition:** _"Pydantic is the validation and serialization library FastAPI is built on. Define a class inheriting from `BaseModel` with type-annotated fields, and Pydantic validates incoming data, coerces types where safe, raises clear errors, and serializes to JSON. Pydantic v2 is written in Rust — 5-50× faster than v1. Critical pattern: **separate input models (`UserCreate`) from output models (`UserPublic`)** — use the `response_model=` argument to filter out fields like passwords from responses. `pydantic-settings` doubles as your env-driven configuration system."_

Pydantic is FastAPI's beating heart. **Pydantic v2 is written in Rust** (5-50x faster than v1) — make sure you're on v2 for any new project.

#### Core Models

```python
from pydantic import BaseModel, Field, EmailStr, field_validator, model_validator
from datetime import datetime
from typing import Annotated

class UserBase(BaseModel):
    name: Annotated[str, Field(min_length=1, max_length=100)]
    email: EmailStr                            # built-in email validation
    age: Annotated[int, Field(ge=0, le=150)] = 18

    # Field-level validator (runs after type coercion)
    @field_validator('name')
    @classmethod
    def name_must_not_be_admin(cls, v: str) -> str:
        if v.lower() == 'admin':
            raise ValueError('Name cannot be "admin"')
        return v.strip().title()                # normalize

    # Model-level validator (runs after all fields validated)
    @model_validator(mode='after')
    def check_consistency(self) -> 'UserBase':
        if self.age < 13 and '@' in self.email:
            # Some made-up rule
            raise ValueError('Children cannot have email accounts')
        return self
```

#### Separate Input vs Output Models (Critical Pattern)

A common mistake: using the same Pydantic model for input and output. Don't.

```python
# Input: what client sends (no id, no timestamps)
class UserCreate(BaseModel):
    name: str
    email: EmailStr
    password: str                              # plaintext from client

# Output: what server returns (no password!)
class UserPublic(BaseModel):
    id: int
    name: str
    email: EmailStr
    created_at: datetime
    # password is NEVER in response — security!

# Internal/DB representation
class UserInDB(UserCreate):
    id: int
    hashed_password: str
    created_at: datetime

# In your endpoint
@app.post("/users", response_model=UserPublic, status_code=201)
async def create_user(user: UserCreate) -> UserPublic:
    # Hash password, save to DB
    db_user = UserInDB(
        id=next_id,
        name=user.name,
        email=user.email,
        hashed_password=hash_password(user.password),
        created_at=datetime.utcnow(),
    )
    # FastAPI auto-filters to only UserPublic fields when returning
    return db_user
```

The `response_model` decorator argument **filters the response** — even if you return extra fields, they're stripped. This prevents accidentally leaking internal fields.

#### Settings Management with Pydantic

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    # All read from env vars: DATABASE_URL, REDIS_URL, etc.
    database_url: str
    redis_url: str = "redis://localhost:6379"
    secret_key: str
    debug: bool = False
    allowed_hosts: list[str] = ["localhost"]

    # Nested config
    stripe_api_key: str
    stripe_webhook_secret: str

# Singleton — load once at startup
settings = Settings()

# In your code
print(settings.database_url)
```

#### Serialization Modes

```python
class User(BaseModel):
    id: int
    created_at: datetime
    password: str = Field(exclude=True)        # never serialize this field

user = User(id=1, created_at=datetime.now(), password="secret")
user.model_dump()                              # → dict (Python types, password excluded)
user.model_dump_json()                         # → JSON string with proper datetime serialization
user.model_dump(exclude={'created_at'})        # exclude on the fly
user.model_dump(include={'id'})                # include only specific fields
```

#### Pydantic v1 → v2 Migration Cheat Sheet

| v1                          | v2                               |
| --------------------------- | -------------------------------- |
| `class Config:`             | `model_config = ConfigDict(...)` |
| `@validator('field')`       | `@field_validator('field')`      |
| `@root_validator`           | `@model_validator(mode='after')` |
| `.dict()`                   | `.model_dump()`                  |
| `.json()`                   | `.model_dump_json()`             |
| `parse_obj()`               | `.model_validate()`              |
| `Optional[X]` (still works) | `X \| None` (preferred)          |

📌 **TLDR:** "Pydantic provides type-driven validation, serialization, and settings management. **Always separate input models (`UserCreate`) from output models (`UserPublic`)** — use `response_model=` to filter responses and prevent leaking fields like passwords. Pydantic v2 is in Rust and dramatically faster. Use `pydantic-settings` for env-driven config. `field_validator` for single fields, `model_validator(mode='after')` for cross-field checks."

---

### 38. Dependency Injection — The Killer Feature

> **📣 Definition:** _"FastAPI's Dependency Injection system lets you declare reusable building blocks — DB sessions, current user, settings, services — that get automatically resolved and injected into endpoints via `Depends(...)`. Dependencies can be functions or classes, can use `yield` for setup/teardown patterns, and compose into a graph that FastAPI resolves per request (with caching so the same dependency runs once even if depended on multiple times). Test substitution is built in: `app.dependency_overrides[real_dep] = mock_dep`. This single feature replaces what you'd build with decorators, middleware, and global state in other frameworks."_

Dependency Injection (DI) is what makes FastAPI feel different. It's not just useful — it's the central organizing principle.

#### The Basics

```python
from fastapi import Depends

# A dependency is just a function (or class)
def get_db():
    db = SessionLocal()
    try:
        yield db                              # yield = return + run cleanup after
    finally:
        db.close()

@app.get("/users")
async def list_users(db = Depends(get_db)):    # FastAPI calls get_db, injects result
    return db.query(User).all()
```

**What FastAPI does behind the scenes:**

1. Sees `db = Depends(get_db)` — recognizes it's a dependency.
2. Calls `get_db()` for each request.
3. The `yield` returns the session to the endpoint.
4. After the endpoint completes (success OR exception), the `finally` block runs → cleanup.
5. The response is returned to the client.

#### Class-Based Dependencies (For State + Validation)

```python
class Pagination:
    def __init__(
        self,
        page: int = Query(1, ge=1),
        size: int = Query(20, ge=1, le=100),
    ):
        self.page = page
        self.size = size
        self.offset = (page - 1) * size

@app.get("/items")
async def list_items(pagination: Pagination = Depends()):
    # Note: Depends() with no arg uses the class itself
    return query.offset(pagination.offset).limit(pagination.size).all()
```

#### Sub-Dependencies (Composition)

Dependencies can depend on other dependencies. FastAPI handles the graph automatically.

```python
def get_db():
    ...

def get_current_user(
    token: str = Depends(oauth2_scheme),
    db = Depends(get_db),                      # ← sub-dependency
):
    user = db.query(User).filter_by(token=token).first()
    if not user:
        raise HTTPException(401)
    return user

def get_admin_user(user: User = Depends(get_current_user)):
    # ← depends on get_current_user, which depends on get_db + oauth2_scheme
    if not user.is_admin:
        raise HTTPException(403)
    return user

@app.delete("/users/{user_id}")
async def delete_user(
    user_id: int,
    admin: User = Depends(get_admin_user),     # full chain executed
    db = Depends(get_db),                      # already cached for this request!
):
    ...
```

**Caching:** within a single request, the same dependency is only called once. So even if `get_db` is depended-on by multiple things in the same endpoint, you get **one DB session** for the whole request.

#### Dependency Scopes (Caching Behavior)

```python
# Default: cached per request
def get_settings():
    return Settings()

@app.get("/")
def root(s1 = Depends(get_settings), s2 = Depends(get_settings)):
    # s1 and s2 are the same instance — get_settings called once

# Disable caching: always re-run
@app.get("/")
def root(s = Depends(get_settings, use_cache=False)):
    ...
```

For app-lifetime singletons (DB pool, Redis client), use `lru_cache`:

```python
from functools import lru_cache

@lru_cache
def get_settings() -> Settings:
    return Settings()                          # called ONCE per process

# Now every request uses the same Settings instance — no re-parsing env vars
```

#### Global Dependencies (Apply to Every Endpoint)

```python
def verify_token(x_token: str = Header()):
    if x_token != "expected-token":
        raise HTTPException(403)

# Applied to ALL endpoints
app = FastAPI(dependencies=[Depends(verify_token)])

# Or per-router
admin_router = APIRouter(
    prefix="/admin",
    dependencies=[Depends(verify_admin_token)],   # only for /admin/*
)
```

#### Why DI is Game-Changing

1. **Test substitution** — swap a real DB with a mock in tests:

```python
def get_db_mock():
    return MockDB()

app.dependency_overrides[get_db] = get_db_mock     # in test setup
```

2. **Reusable logic** — auth, pagination, rate limiting, audit logs all become composable dependencies.

3. **Implicit cleanup** — `try/finally` around `yield` guarantees resource release.

4. **Auto-documentation** — DI parameters show up in OpenAPI as security requirements.

📌 **TLDR:** "FastAPI's DI system is its defining feature. Dependencies are functions/classes injected via `Depends(...)`. Use `yield` for setup/teardown patterns (DB sessions, locks). Sub-dependencies form a graph that FastAPI resolves automatically — same dependency cached per request. Use `app.dependency_overrides` for test substitution. Global dependencies via `FastAPI(dependencies=[...])` apply to all routes — perfect for auth, logging, rate limiting."

---

### 39. Async vs Sync Endpoints (Pitfall Heavy)

> **📣 Definition:** _"FastAPI auto-detects sync vs async endpoints. **`async def`** runs on the event loop directly — perfect when calling async libraries (asyncpg, httpx, aioredis), but **sync I/O inside it (requests, psycopg2, time.sleep) blocks the entire event loop and halts ALL other requests on that worker**. **`def`** runs in a threadpool (default 40 threads) — safe for sync libraries but limited by threadpool size. The rule: pick async when the entire I/O stack is async; pick sync when you're using sync libraries. Don't mix — and never call sync I/O inside `async def`. This is the single biggest cause of production performance bugs in FastAPI."_

This is the #1 source of production bugs in FastAPI. Section 7 covered async fundamentals; this drills into the FastAPI-specific decision.

#### The Two Modes

```python
# ── Async endpoint: runs on the event loop ──────────────────
@app.get("/async")
async def async_endpoint():
    data = await httpx_client.get("...")       # ✅ non-blocking
    return data.json()

# ── Sync endpoint: runs in a threadpool ─────────────────────
@app.get("/sync")
def sync_endpoint():
    data = requests.get("...")                 # blocking, but in a thread
    return data.json()
```

FastAPI **detects which mode** based on `def` vs `async def`:

- `async def` → runs on the event loop directly. Great for async I/O. **DEADLY if you call sync I/O inside.**
- `def` → runs in a thread from a threadpool (default 40 threads). Won't block the event loop, but limited by threadpool size.

#### The Decision Matrix

| Your code calls                                          | Endpoint type                                |
| -------------------------------------------------------- | -------------------------------------------- |
| Async DB driver (asyncpg, motor, aiomysql)               | `async def`                                  |
| `httpx.AsyncClient`, `aioredis`, `aiokafka`              | `async def`                                  |
| Sync libraries (`requests`, `psycopg2`, sync SQLAlchemy) | `def`                                        |
| Mixed sync/async                                         | `def` (safer — sync just runs in threadpool) |
| Pure CPU work (image processing, math)                   | `def` (or offload to multiprocessing)        |
| `time.sleep`, sync file I/O                              | `def`                                        |

#### The #1 Deadly Mistake

```python
# ❌ THE BUG that breaks production
@app.get("/data")
async def get_data():
    # `requests` is SYNC — this BLOCKS the event loop!
    # No other request can be served on this worker until this completes.
    data = requests.get("https://slow-api.com")
    return data.json()
```

If your slow API takes 5 seconds, **the entire worker is paused for 5 seconds**. No other request — even on completely unrelated endpoints — gets served.

```python
# ✅ FIX 1: use the async equivalent
@app.get("/data")
async def get_data():
    async with httpx.AsyncClient() as client:
        data = await client.get("https://slow-api.com")
    return data.json()

# ✅ FIX 2: change to sync def (runs in threadpool, doesn't block loop)
@app.get("/data")
def get_data():
    data = requests.get("https://slow-api.com")
    return data.json()

# ✅ FIX 3: explicitly run in executor
import asyncio

@app.get("/data")
async def get_data():
    loop = asyncio.get_event_loop()
    data = await loop.run_in_executor(
        None,                                  # default threadpool
        lambda: requests.get("https://slow-api.com").json()
    )
    return data
```

#### Detection in the Wild

How to find this bug in a codebase:

```bash
# Look for async functions importing sync libraries
grep -B2 "async def" app/ | grep -E "import (requests|psycopg2|pymongo)"

# Look for time.sleep inside async
grep -A5 "async def" app/ | grep "time.sleep"
```

#### Mixing: Sync DB + Async HTTP

The honest truth: most teams have legacy sync ORM code. The cleanest pattern is to pick one mode per endpoint:

```python
# Legacy sync DB → keep endpoint sync
@app.get("/users/{id}")
def get_user(id: int, db: Session = Depends(get_db)):
    user = db.query(User).get(id)              # sync ORM
    return user

# New endpoints with async DB → make them async
@app.get("/orders/{id}")
async def get_order(id: int, db: AsyncSession = Depends(get_async_db)):
    result = await db.execute(select(Order).where(Order.id == id))
    return result.scalar_one()
```

📌 **TLDR:** "FastAPI auto-detects sync vs async endpoints. **`async def` runs on the event loop — sync I/O inside it (requests, psycopg2, time.sleep) blocks ALL other requests on that worker.** Sync `def` endpoints run in a threadpool (default 40 threads, configurable) — safe but limited concurrency. Pick async for genuinely async stacks (asyncpg + httpx); use sync `def` for legacy/sync libraries. The threadpool isn't unlimited — running 1000 sync requests concurrently against a 40-thread pool means most of them queue up. For CPU-bound work, offload to multiprocessing or a worker queue."

---

### 40. Middleware — Custom and Built-in

> **📣 Definition:** _"FastAPI middleware wraps every request and response — used for cross-cutting concerns: request IDs, timing, logging, CORS, gzip compression, security headers. Define with `@app.middleware('http')` decorator on an async function that takes `(request, call_next)` and returns a response. Order matters: the **last** middleware registered is the **outermost** — first to see requests, last to see responses. Use FastAPI's built-ins (`CORSMiddleware`, `GZipMiddleware`, `TrustedHostMiddleware`, `HTTPSRedirectMiddleware`) before writing your own. For per-route logic, prefer Dependencies over middleware."_

Middleware runs around every request — auth, logging, CORS, compression, request IDs.

#### Custom Middleware

```python
import time
import uuid
from fastapi import Request

@app.middleware("http")
async def add_request_id(request: Request, call_next):
    # Before the endpoint
    request_id = str(uuid.uuid4())
    request.state.request_id = request_id      # available in endpoints

    start = time.time()
    response = await call_next(request)        # calls the next middleware / endpoint
    duration = time.time() - start

    # After the endpoint
    response.headers["X-Request-ID"] = request_id
    response.headers["X-Process-Time"] = f"{duration:.3f}s"

    logger.info(f"{request.method} {request.url.path} → {response.status_code} ({duration:.2f}s)")
    return response

# Multiple middlewares stack — last registered runs OUTERMOST
@app.middleware("http")
async def log_requests(request: Request, call_next):
    response = await call_next(request)
    return response
```

**Execution order** is important and counterintuitive:

```
Request → log_requests → add_request_id → endpoint → add_request_id → log_requests → Response
                  ↑                                            ↓
                  └─────── outermost on response ──────────────┘
```

The **last** middleware registered is the **first** to see the request and the **last** to see the response — like Russian dolls.

#### Built-in Middleware (Use These)

```python
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

# CORS — almost every app needs this
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://yourapp.com"],     # NEVER use ["*"] in production with credentials
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
    max_age=3600,                              # browser caches preflight for 1h
)

# Gzip compression for responses > 1000 bytes
app.add_middleware(GZipMiddleware, minimum_size=1000)

# Reject requests with unexpected Host headers (security)
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["yourapp.com", "*.yourapp.com"],
)

# Force HTTPS in production
if not settings.debug:
    app.add_middleware(HTTPSRedirectMiddleware)
```

#### When to Use Middleware vs Dependencies

| Use case                                                | Middleware     | Dependency |
| ------------------------------------------------------- | -------------- | ---------- |
| Affects EVERY request (logging, request IDs)            | ✅             | ❌         |
| Affects subset of routes (auth on /admin)               | ❌             | ✅         |
| Modifies request/response globally                      | ✅             | ❌         |
| Provides values to endpoints (DB session, current user) | ❌             | ✅         |
| Can short-circuit early (rate limiting)                 | ✅ either      | ✅ either  |
| Has access to typed parameters                          | ❌ raw request | ✅ typed   |

**Rule of thumb:** if you need to provide a value or apply only to some routes → dependency. If it's truly cross-cutting (logging, CORS) → middleware.

📌 **TLDR:** "Middleware wraps every request — use for cross-cutting concerns: request IDs, logging, CORS, gzip, security headers. Last middleware registered runs outermost. Use FastAPI's built-ins (CORSMiddleware, GZipMiddleware, TrustedHostMiddleware) instead of writing your own. For per-route logic (auth, DB sessions, current user) → use dependencies, not middleware."

---

### 41. Background Tasks vs Celery — When to Use What

> **📣 Definition:** _"`BackgroundTasks` runs in the same process after the response is sent — zero infrastructure but tasks are LOST if the worker crashes. Use it for fire-and-forget, non-critical work (cache invalidation, optional emails). Celery is a distributed task queue with durability (Redis/RabbitMQ broker), automatic retries, scheduling via Beat, and horizontal scaling — use it for anything where loss is unacceptable: payments, webhook delivery, billing. **Never use `BackgroundTasks` for money-touching operations.** The right pattern for critical work is the Outbox pattern: write business data + outbox row in one DB transaction, separate worker enqueues to Celery."_

Both run code "after the response is sent." But they're for different problems.

#### `BackgroundTasks` — In-Process Lightweight

```python
from fastapi import BackgroundTasks

def send_welcome_email(email: str):
    # Runs in the SAME process, AFTER the response is returned
    smtp.send(email, "Welcome!")

@app.post("/register")
async def register(user: UserCreate, bg: BackgroundTasks):
    new_user = await create_user(user)
    bg.add_task(send_welcome_email, new_user.email)
    return new_user                            # returned IMMEDIATELY, email sent after
```

**Pros:** zero infrastructure (no Redis/RabbitMQ needed).
**Cons:**

- If the worker crashes after response but before task runs → task is **lost forever**.
- No retries, no scheduling, no visibility.
- Runs in the same process — can't scale horizontally.

#### Celery — Distributed Task Queue

```python
# tasks.py
from celery import Celery

celery_app = Celery(
    "myapp",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/1",
)

@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def send_welcome_email(self, email: str):
    try:
        smtp.send(email, "Welcome!")
    except SMTPException as e:
        # Retry with exponential backoff
        raise self.retry(exc=e, countdown=2 ** self.request.retries)

# main.py
@app.post("/register")
async def register(user: UserCreate):
    new_user = await create_user(user)
    send_welcome_email.delay(new_user.email)   # enqueued in Redis, processed by Celery worker
    return new_user
```

**Pros:** durable (survives crashes), retries, scheduling (`celery beat`), horizontal scaling, monitoring (Flower).
**Cons:** infrastructure (Redis/RabbitMQ), deployment complexity, harder local dev.

#### Decision Matrix

| Use case                                 | Tool                                                        |
| ---------------------------------------- | ----------------------------------------------------------- |
| Send a single welcome email after signup | `BackgroundTasks` (acceptable to lose)                      |
| Process a payment                        | **NEVER use BackgroundTasks** — use Celery + DB persistence |
| Generate a PDF report                    | Celery (can take minutes)                                   |
| Cache invalidation                       | `BackgroundTasks` (idempotent, retry on next request)       |
| Webhook delivery to external service     | Celery with retries                                         |
| Scheduled jobs (daily reports)           | Celery Beat                                                 |
| Image processing                         | Celery with separate worker pool                            |

#### The Right Pattern for Critical Tasks

```python
# Don't enqueue a task → write to DB → return.
# DO: write to DB → enqueue → return. Use the Outbox pattern.

@app.post("/orders")
async def create_order(order: OrderCreate, db: AsyncSession = Depends(get_db)):
    async with db.begin():
        new_order = Order(**order.dict())
        db.add(new_order)

        # Outbox row in same transaction
        outbox = OutboxEvent(
            event_type='order.created',
            payload=new_order.to_dict(),
            status='pending',
        )
        db.add(outbox)
        # commit happens at end of `async with db.begin()`

    return new_order

# Separate worker polls outbox and dispatches to Celery
```

This guarantees: **no order in DB without a corresponding pending event**, and **no event published without a committed order**.

📌 **TLDR:** "`BackgroundTasks` is for in-process, fire-and-forget work that's OK to lose if the server crashes (cache invalidation, non-critical emails). For anything where loss is unacceptable (payments, webhook delivery, billing), use Celery — durable queue, retries, scheduling, horizontal scaling. Critical pattern: combine Celery with the Outbox pattern — write business data + outbox row in same DB transaction, separate worker drains outbox and enqueues the Celery task. Never use `BackgroundTasks` for money-touching operations."

---

### 42. Authentication & Authorization (OAuth2 + JWT)

> **📣 Definition:** _"The standard FastAPI auth pattern is OAuth2 password flow with JWT bearer tokens: `OAuth2PasswordBearer` extracts the token from the `Authorization` header, `python-jose` encodes/decodes JWTs, and `passlib[bcrypt]` hashes passwords. Production needs short-lived access tokens (15-30 min) plus database-backed refresh tokens (7-30 days, rotated on use) so logout/revocation actually works. Build RBAC as dependency factories — `require_role('admin')` returns a Depends-able. JWT pitfalls: never store secrets in the payload, pin the algorithm, plan for revocation."_

#### The Standard FastAPI Pattern

```python
from datetime import datetime, timedelta
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt   # ⚠️ python-jose is unmaintained (last release 2022)
# Modern alternatives (nearly identical API):
#   PyJWT:   pip install PyJWT   → import jwt; jwt.encode()/jwt.decode()
#   authlib: pip install authlib → from authlib.jose import jwt
from passlib.context import CryptContext

SECRET_KEY = settings.secret_key
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/token")

# ── Password helpers ────────────────────────────────────────
def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

# ── JWT helpers ─────────────────────────────────────────────
def create_access_token(data: dict, expires_delta: timedelta = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

# ── Login endpoint ──────────────────────────────────────────
@app.post("/auth/token")
async def login(form: OAuth2PasswordRequestForm = Depends()):
    user = get_user_by_email(form.username)        # form.username = email
    if not user or not verify_password(form.password, user.hashed_password):
        raise HTTPException(401, "Invalid credentials")
    token = create_access_token({"sub": str(user.id)})
    return {"access_token": token, "token_type": "bearer"}

# ── Protected endpoint dependency ───────────────────────────
async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = get_user_by_id(int(user_id))
    if user is None:
        raise credentials_exception
    return user

# ── Use in protected routes ─────────────────────────────────
@app.get("/me")
async def me(current_user: User = Depends(get_current_user)):
    return current_user
```

#### Role-Based Access Control (RBAC)

```python
def require_role(role: str):
    """Dependency factory — returns a dependency that checks for a role."""
    async def role_checker(user: User = Depends(get_current_user)) -> User:
        if role not in user.roles:
            raise HTTPException(403, f"Requires role: {role}")
        return user
    return role_checker

@app.delete("/users/{id}")
async def delete_user(
    id: int,
    user: User = Depends(require_role("admin")),
):
    ...

@app.get("/reports")
async def get_reports(
    user: User = Depends(require_role("manager")),
):
    ...
```

#### Refresh Tokens (Production Pattern)

Access tokens are short-lived (15-30 min). Refresh tokens are long-lived (7-30 days), stored in the DB so they can be revoked.

```python
@app.post("/auth/refresh")
async def refresh(refresh_token: str, db: AsyncSession = Depends(get_db)):
    # Lookup in DB — revocation requires DB check
    record = await db.execute(
        select(RefreshToken).where(RefreshToken.token == refresh_token)
    )
    rt = record.scalar_one_or_none()
    if not rt or rt.revoked or rt.expires_at < datetime.utcnow():
        raise HTTPException(401, "Invalid refresh token")

    # Rotate: invalidate old, issue new
    rt.revoked = True
    new_refresh = create_refresh_token(rt.user_id)
    new_access = create_access_token({"sub": str(rt.user_id)})

    return {"access_token": new_access, "refresh_token": new_refresh}
```

#### JWT Pitfalls

1. **Don't store sensitive data** — JWTs are base64-encoded, not encrypted. Anyone can decode the payload.
2. **Use short expiry** for access tokens (15-30 min). Use refresh tokens for "stay logged in."
3. **Logout is hard** — JWTs are stateless. Either: short expiry + refresh token revocation, OR maintain a token blocklist in Redis.
4. **Algorithm confusion attack** — never accept `alg=none`. Pin the algorithm in your `decode` call.
5. **Use HS256 with strong secret** OR **RS256/ES256** with key pairs (better for distributed systems where multiple services verify tokens).

📌 **TLDR:** "OAuth2 + JWT is the standard. Use `OAuth2PasswordBearer` + `jose` for JWT + `passlib[bcrypt]` for password hashing. Short-lived access tokens (15-30 min) + database-backed refresh tokens (7-30 days, rotated on use) for production. Build RBAC as dependency factories — `require_role('admin')` returns a dependency. Watch for JWT pitfalls: never store secrets, pin the algorithm, plan for revocation (refresh token rotation OR Redis blocklist)."

---

### 43. Database Integration — SQLAlchemy 2.0 + Alembic

> **📣 Definition:** _"FastAPI ships without an ORM — SQLAlchemy 2.0 (async) with the typed `Mapped[type]` syntax is the de-facto choice. Inject sessions via a Dependency that uses `yield` for cleanup (commit on success, rollback on exception). Use `selectinload` (separate query — equivalent to Django's `prefetch_related`) for one-to-many and `joinedload` (JOIN — equivalent to `select_related`) for one-to-one. **Never** use `Base.metadata.create_all()` in production — Alembic is mandatory for tracked, reversible migrations. Connection pool tuning: `pool_size × workers ≤ DB max_connections`. Use PgBouncer in front of Postgres for high-scale apps."_

FastAPI doesn't ship with an ORM. SQLAlchemy 2.0 (with the new `Mapped` typed syntax) is the de-facto choice.

#### Async SQLAlchemy 2.0 Setup

```python
# database.py
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

engine = create_async_engine(
    settings.database_url,                     # "postgresql+asyncpg://user:pass@host/db"
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,                        # check connection alive before use
    echo=False,                                # True = log all SQL
)

AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

# Modern typed models
class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    hashed_password: Mapped[str]
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
```

#### DB Session Dependency

```python
async def get_db() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        try:
            yield session
        except Exception:
            await session.rollback()
            raise
        # session.close() happens automatically via async with
```

#### CRUD Patterns

```python
from sqlalchemy import select, update, delete

# Read
@app.get("/users/{id}")
async def get_user(id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.id == id))
    user = result.scalar_one_or_none()
    if user is None:
        raise HTTPException(404)
    return user

# Create
@app.post("/users", status_code=201)
async def create_user(data: UserCreate, db: AsyncSession = Depends(get_db)):
    user = User(email=data.email, hashed_password=hash_password(data.password))
    db.add(user)
    await db.commit()
    await db.refresh(user)                     # populate auto-generated fields (id)
    return user

# Update
@app.patch("/users/{id}")
async def update_user(id: int, data: UserUpdate, db: AsyncSession = Depends(get_db)):
    await db.execute(
        update(User).where(User.id == id).values(**data.dict(exclude_unset=True))
    )
    await db.commit()
    return await get_user(id, db)

# Eager loading (the equivalent of select_related/prefetch_related)
from sqlalchemy.orm import selectinload, joinedload

@app.get("/orders/{id}")
async def get_order_with_items(id: int, db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order)
        .where(Order.id == id)
        .options(
            joinedload(Order.user),            # JOIN — for one-to-one / many-to-one
            selectinload(Order.items),         # separate query — for one-to-many
        )
    )
    return result.unique().scalar_one()
```

#### Migrations with Alembic

```bash
# Initial setup
alembic init -t async migrations

# Create a migration
alembic revision --autogenerate -m "add user table"

# Apply migrations
alembic upgrade head

# Rollback
alembic downgrade -1
```

**Tech-lead concern: never use `Base.metadata.create_all()` in production.** That's only for tests/local dev. Production migrations MUST go through Alembic for tracked, reversible changes.

#### Connection Pooling (Critical for Production)

| Setting         | Value            | Why                                             |
| --------------- | ---------------- | ----------------------------------------------- |
| `pool_size`     | 10-20 per worker | base connections kept open                      |
| `max_overflow`  | 5-10             | extra connections for traffic spikes            |
| `pool_pre_ping` | `True`           | detects stale connections (PG closes idle ones) |
| `pool_recycle`  | 3600             | reset connections every hour to avoid timeouts  |

If you have 4 workers × `pool_size=20` = 80 connections. Postgres default `max_connections=100`. Plan accordingly — use **PgBouncer** as a connection pooler in front of Postgres for high-scale apps.

📌 **TLDR:** "Use SQLAlchemy 2.0 async with `Mapped[type]` typed syntax (Pydantic-friendly). Inject sessions via dependency with `yield` for cleanup. Use `selectinload` (separate query, like Django's `prefetch_related`) for one-to-many and `joinedload` (JOIN, like `select_related`) for one-to-one. Migrate with Alembic — never `create_all()` in production. Tune the connection pool: `pool_size` × workers ≤ DB max_connections. Use PgBouncer for scale."

---

### 44. WebSockets & Server-Sent Events

> **📣 Definition:** _"FastAPI supports WebSockets natively for bidirectional real-time communication (chat, dashboards, collaborative editing) via `@app.websocket('/ws')` decorator. The production challenge: with multiple Uvicorn workers, in-memory ConnectionManagers can't broadcast across workers — use Redis pub/sub as a cross-worker message bus, or battle-tested tools like Django Channels/Centrifugo. For one-way server-push (notifications, AI token streaming), Server-Sent Events via `StreamingResponse` with `text/event-stream` is simpler and more reliable than WebSockets and works through standard HTTP infrastructure."_

#### WebSockets (Bidirectional Real-Time)

```python
from fastapi import WebSocket, WebSocketDisconnect

class ConnectionManager:
    def __init__(self):
        self.active: dict[str, WebSocket] = {}

    async def connect(self, ws: WebSocket, user_id: str):
        await ws.accept()
        self.active[user_id] = ws

    def disconnect(self, user_id: str):
        self.active.pop(user_id, None)

    async def send_to(self, user_id: str, message: dict):
        if ws := self.active.get(user_id):
            await ws.send_json(message)

    async def broadcast(self, message: dict):
        for ws in self.active.values():
            await ws.send_json(message)

manager = ConnectionManager()

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(ws: WebSocket, user_id: str):
    await manager.connect(ws, user_id)
    try:
        while True:
            data = await ws.receive_json()
            # Process message — e.g., chat, notifications
            await manager.broadcast({"from": user_id, "data": data})
    except WebSocketDisconnect:
        manager.disconnect(user_id)
```

**Production caveat:** `ConnectionManager` above stores connections in process memory. With multiple Uvicorn workers, a user connected to worker A can't receive messages from worker B. Solution: **Redis pub/sub** as the cross-worker message bus.

```python
# Subscribe in one worker, publish from anywhere
import redis.asyncio as redis

r = redis.from_url("redis://localhost")

# Publisher (any worker)
await r.publish(f"user:{user_id}", json.dumps(message))

# Subscriber (each worker maintains its own)
pubsub = r.pubsub()
await pubsub.subscribe("user:*")
async for message in pubsub.listen():
    if user_id := extract_user_id(message['channel']):
        await manager.send_to(user_id, json.loads(message['data']))
```

For real-world scale, consider **Django Channels** or **Centrifugo** (battle-tested) over rolling your own.

#### Server-Sent Events (One-Way Streaming)

For **one-way** server-to-client streams (notifications, logs, AI token streaming), SSE is simpler than WebSockets:

```python
from fastapi.responses import StreamingResponse
import asyncio

async def event_generator():
    for i in range(100):
        yield f"data: {json.dumps({'count': i})}\n\n"
        await asyncio.sleep(1)

@app.get("/events")
async def events():
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

Used heavily in AI applications for streaming LLM tokens.

📌 **TLDR:** "FastAPI supports WebSockets natively for bidirectional real-time. Production challenge: workers don't share memory — use Redis pub/sub as the cross-worker bus, or use battle-tested tools like Django Channels or Centrifugo. For one-way server-push (notifications, AI token streaming), Server-Sent Events (`StreamingResponse` with `text/event-stream`) is simpler and more reliable than WebSockets."

---

### 45. Testing — TestClient, AsyncClient, Fixtures

> **📣 Definition:** _"FastAPI testing uses `httpx.AsyncClient` with `ASGITransport(app=app)` — the modern async equivalent of `TestClient` (which is sync). Pytest fixtures handle DB setup/teardown per test, and `app.dependency_overrides[real_dep] = mock_dep` swaps real DBs and external services for test doubles — this is the cleanest pattern for unit testing a DI-heavy app. Always test against a real Postgres in Docker (not SQLite — they have different behaviors). Mock external APIs at the boundary, either via `unittest.mock.patch` or by replacing the service-class dependency."_

#### Test Setup

```python
# conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

from main import app
from database import Base, get_db

# Use a separate test database
TEST_DB_URL = "postgresql+asyncpg://test:test@localhost/test_db"
test_engine = create_async_engine(TEST_DB_URL)
TestSession = async_sessionmaker(test_engine, expire_on_commit=False)


@pytest_asyncio.fixture
async def db_session():
    """Fresh DB per test, rolled back at the end."""
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    async with TestSession() as session:
        yield session
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def client(db_session):
    """Override get_db with the test session."""
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

    app.dependency_overrides.clear()
```

#### Writing Tests

```python
# test_users.py
import pytest

@pytest.mark.asyncio
async def test_create_user(client):
    response = await client.post("/users", json={
        "email": "test@example.com",
        "password": "secret",
    })
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "password" not in data                   # password should NEVER leak

@pytest.mark.asyncio
async def test_get_nonexistent_user(client):
    response = await client.get("/users/99999")
    assert response.status_code == 404

@pytest.mark.asyncio
async def test_authentication_required(client):
    response = await client.get("/me")              # no auth header
    assert response.status_code == 401

@pytest.mark.asyncio
async def test_authenticated_endpoint(client, db_session):
    # Create user + login
    user = User(email="t@e.com", hashed_password=hash_password("pw"))
    db_session.add(user)
    await db_session.commit()

    login_resp = await client.post("/auth/token", data={"username": "t@e.com", "password": "pw"})
    token = login_resp.json()["access_token"]

    response = await client.get("/me", headers={"Authorization": f"Bearer {token}"})
    assert response.status_code == 200
```

#### Mocking External Services

```python
from unittest.mock import patch

@pytest.mark.asyncio
async def test_payment_with_mocked_stripe(client):
    with patch("services.payment.stripe.PaymentIntent.create") as mock_stripe:
        mock_stripe.return_value = MagicMock(id="pi_mock_123", status="succeeded")

        response = await client.post("/payments", json={"amount": 5000})
        assert response.status_code == 201
        mock_stripe.assert_called_once()

# Better: dependency overrides for service classes
class MockPaymentService:
    async def charge(self, amount):
        return {"id": "mock_123", "status": "ok"}

app.dependency_overrides[get_payment_service] = lambda: MockPaymentService()
```

📌 **TLDR:** "Use `httpx.AsyncClient` with `ASGITransport(app=app)` for async test clients (the modern replacement for `TestClient` which is sync). Override dependencies via `app.dependency_overrides[real_dep] = mock_dep` to swap real DBs/services with test doubles. Use a separate test database with per-test setup/teardown via fixtures. Mock external APIs at the boundary — either via `unittest.mock.patch` or by replacing the service-class dependency."

---

### 46. Project Structure for Large Apps

> **📣 Definition:** _"FastAPI's official tutorial uses a flat layout that doesn't scale beyond a small app. For real services, organize by **domain module** — `auth/`, `users/`, `orders/`, `payments/` — each with its own `router.py` (HTTP layer), `schemas.py` (Pydantic), `models.py` (SQLAlchemy), `service.py` (business logic, framework-agnostic), and `repository.py` (DB access). The router is a thin layer that translates HTTP to service calls; services contain zero FastAPI imports so they're testable without a web server. Use `lifespan` (not deprecated startup/shutdown events) for resource setup/teardown."_

The single biggest predictor of FastAPI codebase health is structure. The official tutorial's flat layout doesn't scale.

#### Recommended Layout (Domain-Driven)

```
myapp/
├── alembic/                          # migrations
├── pyproject.toml
├── README.md
├── docker-compose.yml
├── Dockerfile
└── src/
    └── myapp/
        ├── __init__.py
        ├── main.py                   # FastAPI app instance, mount routers
        ├── config.py                 # Pydantic Settings
        ├── database.py               # engine, session factory, get_db
        ├── dependencies.py           # shared deps (current_user, etc.)
        ├── exceptions.py             # custom exception classes
        ├── middleware.py             # custom middleware
        │
        ├── auth/                     # 🟢 Domain module
        │   ├── __init__.py
        │   ├── router.py             # APIRouter with endpoints
        │   ├── schemas.py            # Pydantic models
        │   ├── models.py             # SQLAlchemy models
        │   ├── service.py            # Business logic (no FastAPI here)
        │   ├── repository.py         # DB access
        │   ├── dependencies.py       # auth-specific deps
        │   └── tests/
        │
        ├── users/                    # 🟢 Domain module
        │   ├── __init__.py
        │   ├── router.py
        │   ├── schemas.py
        │   ├── models.py
        │   ├── service.py
        │   └── tests/
        │
        ├── orders/                   # 🟢 Domain module
        │   ├── ...
        │
        └── payments/                 # 🟢 Domain module
            ├── ...
```

#### `main.py` — Composition Root

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from .config import settings
from .database import engine
from .auth.router import router as auth_router
from .users.router import router as users_router
from .orders.router import router as orders_router

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: warm up connections
    print("Starting up...")
    yield
    # Shutdown: dispose engine, flush logs
    await engine.dispose()
    print("Shutting down...")

app = FastAPI(
    title="My App",
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs" if settings.debug else None,    # disable in prod
)

# Middleware
app.add_middleware(CORSMiddleware, ...)

# Routers
app.include_router(auth_router, prefix="/auth", tags=["auth"])
app.include_router(users_router, prefix="/users", tags=["users"])
app.include_router(orders_router, prefix="/orders", tags=["orders"])

@app.get("/health")
async def health():
    return {"status": "ok"}
```

#### Layered Architecture (per Domain Module)

```python
# users/service.py — Pure business logic, no FastAPI imports!
class UserService:
    def __init__(self, repo: UserRepository, mailer: Mailer):
        self.repo = repo
        self.mailer = mailer

    async def register(self, email: str, password: str) -> User:
        if await self.repo.exists(email):
            raise UserAlreadyExists()
        user = User(email=email, hashed_password=hash_password(password))
        await self.repo.add(user)
        await self.mailer.send_welcome(email)
        return user

# users/router.py — Thin layer that translates HTTP ↔ Service
@router.post("/", response_model=UserPublic, status_code=201)
async def register(
    data: UserCreate,
    service: UserService = Depends(get_user_service),
):
    try:
        return await service.register(data.email, data.password)
    except UserAlreadyExists:
        raise HTTPException(409, "Email already registered")
```

**Why this layering matters:**

- Service layer is **framework-agnostic** — you can switch to Flask, Django, or even a CLI without touching it.
- Tests can target the service directly without HTTP.
- Domain logic doesn't know about HTTP status codes — that's the router's job.

#### `lifespan` — Replacing Startup/Shutdown Events

The old `@app.on_event("startup")` is **deprecated**. Use `lifespan`:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # ── Startup ─────────────────────────────────────
    redis_client = await create_redis_pool()
    app.state.redis = redis_client
    app.state.kafka_producer = await create_kafka_producer()

    yield                                           # app runs

    # ── Shutdown ────────────────────────────────────
    await redis_client.close()
    await app.state.kafka_producer.stop()

app = FastAPI(lifespan=lifespan)
```

📌 **TLDR:** "Don't use the flat tutorial structure for real apps. Organize by **domain module** (`auth/`, `users/`, `orders/`) — each with its own router, schemas, models, service, and repository. Keep services framework-agnostic — no FastAPI imports in business logic. The router is a thin HTTP-translation layer. Use `lifespan` (not the deprecated startup/shutdown events) for resource setup/teardown. Disable `/docs` in production."

---

### 47. Production Deployment

> **📣 Definition:** _"Production FastAPI runs on Gunicorn with UvicornWorker (or pure Uvicorn) — `2×CPU+1` workers per instance for I/O-bound apps. Sit it behind Nginx or Traefik for TLS, static files, and reverse proxy logging. Mandatory production stack pieces: PgBouncer for Postgres connection pooling, Redis for caching and Celery, two health endpoints (`/health` for liveness, `/ready` for readiness with DB+Redis checks), Sentry for errors, OpenTelemetry for distributed tracing, Prometheus for metrics. Always disable `/docs` in production for security and run as a non-root user in the container."_

```bash
# Development
uvicorn myapp.main:app --reload --port 8000

# Production — multiple workers
gunicorn myapp.main:app -w 4 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000

# Or pure uvicorn
uvicorn myapp.main:app --workers 4 --host 0.0.0.0 --port 8000
```

**Worker count rule of thumb:**

- I/O-bound (most APIs): `2 × CPU_cores + 1`
- CPU-bound: `CPU_cores`
- Mixed: start with `CPU_cores`, scale based on metrics

#### Production Stack

```
                        ┌─────────────────────────┐
                        │  Load Balancer (LB)     │
                        │  - TLS termination      │
                        │  - Rate limit (Cloudflare)
                        └──────────┬──────────────┘
                                   │
                        ┌──────────▼──────────────┐
                        │  Nginx / Traefik         │
                        │  - reverse proxy         │
                        │  - static files          │
                        │  - request logging       │
                        └──────────┬──────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │  Uvicorn W1    │  │  Uvicorn W2    │  │  Uvicorn Wn    │
     │  (FastAPI app) │  │  (FastAPI app) │  │  (FastAPI app) │
     └────────┬───────┘  └────────┬───────┘  └────────┬───────┘
              │                   │                    │
              └───────────────────┼────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
     ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
     │   Postgres     │  │     Redis      │  │     Celery     │
     │  (PgBouncer    │  │ (cache + queue)│  │   (workers)    │
     │   in front)    │  │                │  │                │
     └────────────────┘  └────────────────┘  └────────────────┘
```

#### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# Install uv (fast Python package manager)
RUN pip install --no-cache-dir uv

# Install dependencies
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# Copy app
COPY src/ ./src/

# Non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8000
CMD ["uv", "run", "gunicorn", "myapp.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

#### Health Checks

```python
@app.get("/health")                            # liveness — am I alive?
async def health():
    return {"status": "ok"}

@app.get("/ready")                             # readiness — can I take traffic?
async def ready(db: AsyncSession = Depends(get_db), redis = Depends(get_redis)):
    try:
        await db.execute(text("SELECT 1"))
        await redis.ping()
        return {"status": "ready"}
    except Exception:
        raise HTTPException(503, "Not ready")
```

Kubernetes `livenessProbe` hits `/health` (restart on failure); `readinessProbe` hits `/ready` (remove from LB on failure).

#### Observability (Production Non-Negotiables)

```python
# Structured JSON logging
import logging, json
from pythonjsonlogger import jsonlogger

handler = logging.StreamHandler()
handler.setFormatter(jsonlogger.JsonFormatter())

# OpenTelemetry tracing
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
FastAPIInstrumentor.instrument_app(app)        # auto-traces every request

# Prometheus metrics
from prometheus_fastapi_instrumentator import Instrumentator
Instrumentator().instrument(app).expose(app)   # /metrics endpoint

# Sentry for errors
import sentry_sdk
sentry_sdk.init(dsn=settings.sentry_dsn)
```

📌 **TLDR:** "Production = Gunicorn + UvicornWorker (or pure Uvicorn) with `2×CPU+1` workers for I/O-bound apps, behind Nginx/Traefik for static files and TLS. Use `lifespan` for startup/shutdown. Two health checks: `/health` (liveness) and `/ready` (checks DB/Redis). Observability stack: structured JSON logs (Loki/ELK), OpenTelemetry tracing, Prometheus metrics, Sentry for errors. Disable `/docs` in production. Use PgBouncer in front of Postgres for connection pooling at scale."

---

### 48. Performance Optimization & Pitfalls

> **📣 Definition:** _"FastAPI is fast by default, but production performance dies from a small set of common issues — in order of frequency: sync I/O hidden inside `async def` endpoints (blocks the event loop), N+1 queries, missing connection pooling, no caching, large unpaginated responses, synchronous CPU work blocking the event loop, Pydantic v1 instead of v2, and missing `uvicorn[standard]` (which installs uvloop and httptools). Profile before optimizing — `yappi` for async profiling or per-request timing middleware. Cache hot reads in Redis (cross-worker) since `lru_cache` is per-process and goes stale across instances."_

#### Top 10 FastAPI Performance Issues (in order of frequency)

1. **Sync I/O in `async def` endpoints** — covered in 16.5. Single biggest perf killer.
2. **N+1 queries** — same problem as Django; use `selectinload`/`joinedload`.
3. **No connection pooling** — every request opens a new DB connection. Always use a pool.
4. **Large response payloads** — enable GZip middleware, paginate, project columns (don't `SELECT *`).
5. **Synchronous CPU work in event loop** — JSON parsing 10MB payloads, image processing. Offload to threadpool or worker.
6. **Pydantic v1** — upgrade to v2 (Rust). 5-50x faster validation.
7. **No caching** — Redis cache for expensive reads, response caching for static endpoints.
8. **Logging overhead** — sync logging on every request adds latency. Use async logging or batched.
9. **Unbounded query results** — `User.objects.all()` style on million-row tables. Always paginate.
10. **Default uvloop disabled** — explicitly use `uvicorn[standard]` to get uvloop + httptools.

#### Profiling Tips

```python
# Async profiler
import yappi
yappi.start()
# ... run requests ...
yappi.get_func_stats().print_all()

# Per-request timing
@app.middleware("http")
async def timing(request, call_next):
    start = time.time()
    response = await call_next(request)
    response.headers["X-Time"] = f"{(time.time() - start) * 1000:.0f}ms"
    return response
```

#### Caching Patterns

```python
from functools import lru_cache
from cachetools import TTLCache

# In-memory per-process (fast but not shared across workers)
@lru_cache(maxsize=1000)
def get_static_data(key: str):
    return expensive_computation(key)

# TTL cache — expires after N seconds
cache = TTLCache(maxsize=10000, ttl=300)

# Redis (shared across workers)
async def get_user_cached(user_id: int, redis = Depends(get_redis), db = Depends(get_db)):
    cache_key = f"user:{user_id}"
    if cached := await redis.get(cache_key):
        return User.model_validate_json(cached)

    user = await db.get(User, user_id)
    await redis.setex(cache_key, 300, user.model_dump_json())
    return user
```

📌 **TLDR:** "Top FastAPI perf issues: sync I/O in async endpoints, N+1 queries, no connection pooling, no caching, and Pydantic v1. Use `uvicorn[standard]` for uvloop + httptools. Profile with yappi or per-request middleware timing. Cache hot reads in Redis (cross-worker) — `lru_cache` is per-process so stale data across workers is a risk."

---

### 49. FastAPI vs Django vs Flask — When to Choose What

> **📣 Definition:** _"Choose **FastAPI** for API-first services where async I/O matters, type safety is valuable, and you want auto-generated OpenAPI docs — best for microservices, real-time backends, and ML/AI APIs. Choose **Django** for monolithic apps where the admin UI saves weeks, you need batteries (auth/sessions/forms/ORM/migrations), and the app does meaningful server-side rendering — best for content-heavy products and internal tools. Choose **Flask** for tiny apps or legacy code where you need maximum simplicity. Common modern pattern: Django monolith for the main product + FastAPI microservices for performance-critical or async-heavy paths."_

This question comes up in every architect interview.

| Aspect                  | FastAPI                | Django                | Flask         |
| ----------------------- | ---------------------- | --------------------- | ------------- |
| **Speed**               | Very high              | Medium                | Medium        |
| **Async**               | Native                 | Bolted-on (improving) | Limited       |
| **Built-in admin**      | ❌                     | ✅ Killer feature     | ❌            |
| **ORM**                 | None (BYOORM)          | Built-in (great)      | None          |
| **Migrations**          | Alembic (separate)     | Built-in              | Alembic       |
| **Auth**                | DIY/libraries          | Built-in (great)      | DIY/libraries |
| **API focus**           | API-first              | Full-stack web        | Minimal       |
| **Type safety**         | First-class            | Limited               | Limited       |
| **Auto docs (OpenAPI)** | ✅ Free                | ❌ DRF needed         | ❌            |
| **Maturity**            | Newer (2018)           | Very mature (2005)    | Mature (2010) |
| **Community size**      | Large + growing        | Massive               | Massive       |
| **Learning curve**      | Low if you know typing | Medium                | Very low      |

#### Decision Tree

```
Need an admin UI for non-technical users? → Django
Building primarily a RESTful API for SPA/mobile? → FastAPI
Async I/O is critical (real-time, high concurrency)? → FastAPI
Tiny app, <5 endpoints? → Flask (or FastAPI — both fine)
Complex domain with tons of CRUD + auth + sessions? → Django
Microservice that talks to other services? → FastAPI
Full-stack web app with templates + forms? → Django
```

#### When NOT to Choose FastAPI

- You need Django's batteries (admin, sessions, forms) and won't have time to build them.
- Team has zero typing experience and you need rapid prototyping.
- Heavy server-side rendering (Django templates beat anything FastAPI offers natively).

📌 **TLDR:** "FastAPI for API-first, async-heavy services where you want type safety and auto-docs. Django for monolithic apps with admin UI, complex domain models, and full-stack web (templates + auth + sessions). Flask for simple/legacy. The most common modern pattern: Django for the main monolith + FastAPI microservices for performance-critical endpoints."

---

### 50. Tech-Lead Interview Q&A

> **📣 Definition:** _"This is the curated set of questions that come up most often in FastAPI tech-lead interviews — async vs sync discipline, dependency injection internals, project structure, auth patterns, BackgroundTasks vs Celery, N+1 prevention with SQLAlchemy, testing strategy, FastAPI-vs-Django tradeoffs, scaling to 10k RPS, and the request lifecycle. The pattern in good answers: name the concept, give a concrete example, acknowledge the trade-off. Generic answers signal junior; trade-off-aware answers with production context signal senior."_

**Q1: "What's the difference between `def` and `async def` endpoints in FastAPI?"**

> "`async def` runs on the event loop directly — perfect for async I/O (asyncpg, httpx, aioredis). `def` runs in a threadpool (default 40 threads) — safe for sync libraries (requests, psycopg2). The deadly mistake: calling sync I/O inside `async def` blocks the entire event loop, halting all requests on that worker. Detection rule: if you're using `requests` or sync ORM, use `def`. If you're using `httpx.AsyncClient` and asyncpg, use `async def`. Don't mix."

**Q2: "How does FastAPI's dependency injection work?"**

> "Dependencies are functions or classes injected via `Depends(...)`. FastAPI builds a dependency graph, resolves it for each request, and caches results within that request — same `get_db` called multiple times yields the same session. Use `yield` for setup/teardown patterns. Sub-dependencies compose naturally — `get_admin_user` depends on `get_current_user` depends on `get_db`. For tests, override with `app.dependency_overrides[real] = mock`. It's basically pytest fixtures for production."

**Q3: "How would you structure a large FastAPI codebase?"**

> "Domain-driven layout: top-level modules per domain (`auth/`, `users/`, `orders/`), each with `router.py` (HTTP layer), `schemas.py` (Pydantic), `models.py` (SQLAlchemy), `service.py` (business logic, framework-agnostic), and `repository.py` (DB access). The service layer should have zero FastAPI imports — that lets you switch frameworks without rewriting business logic and makes testing straightforward. The router is a thin layer that translates HTTP to service calls."

**Q4: "How do you handle authentication in FastAPI?"**

> "OAuth2 password flow with JWT. `OAuth2PasswordBearer` for token extraction, `python-jose` (or `PyJWT`/`authlib` — note `jose` is unmaintained) for JWT encode/decode, `passlib[bcrypt]` for password hashing. Short-lived access tokens (15-30 min) plus database-backed refresh tokens (7-30 days, rotated on use) for logout/revocation. Build RBAC as dependency factories — `require_role('admin')` returns a dependency. For service-to-service, prefer mTLS or RS256-signed JWTs verified by public key."

**Q5: "What's the difference between BackgroundTasks and Celery?"**

> "`BackgroundTasks` runs in the same process after the response is returned — zero infrastructure but lost on crash. Use for non-critical fire-and-forget work (cache invalidation, non-essential emails). Celery is a durable distributed queue (Redis/RabbitMQ broker) with retries, scheduling, and horizontal scaling — use for anything where loss is unacceptable: payment processing, webhook delivery, billing. The right pattern for critical work is the Outbox pattern: write business data + outbox row in same DB transaction, separate worker drains outbox and enqueues to Celery."

**Q6: "How do you avoid N+1 queries in FastAPI?"**

> "Same problem as Django, different ORM. SQLAlchemy 2.0: use `joinedload` for one-to-one and many-to-one (single JOIN query), `selectinload` for one-to-many and many-to-many (separate `IN` query). Inspect generated SQL with `echo=True` on the engine in dev, or use SQLAlchemy event hooks to log queries per request. Add a middleware that asserts query count under a threshold in tests."

**Q7: "How do you test a FastAPI app?"**

> "Use `httpx.AsyncClient` with `ASGITransport(app=app)` — the async equivalent of `TestClient`. Pytest fixtures for DB session (per-test setup/teardown) and authenticated client. Override dependencies via `app.dependency_overrides` to swap real DBs for test DBs and mock external services. Run tests against a real Postgres in Docker, not SQLite — they have different behaviors. Aim for tests at the router level (integration) plus pure-unit tests on the service layer."

**Q8: "When would you choose FastAPI over Django?"**

> "FastAPI for API-first services, especially async-heavy (real-time, lots of concurrent I/O), where I want first-class type safety and free OpenAPI docs. Django for monolithic apps where the admin UI saves weeks of CRUD building, where I need batteries (auth, sessions, forms, ORM, migrations) included. The hybrid pattern is common: Django monolith for the main product + FastAPI microservices for performance-critical or async-heavy endpoints."

**Q9: "How do you scale a FastAPI app to 10k RPS?"**

> "Horizontal: Gunicorn with UvicornWorker, `2×CPU+1` workers per instance, multiple instances behind a load balancer. Make sure async endpoints actually call async I/O (no sync I/O hidden inside). Connection pooling with PgBouncer in front of Postgres. Redis cache for hot reads. Observability with Prometheus + OpenTelemetry from day one — you can't optimize what you can't measure. CDN for static assets. For genuinely high concurrent connections (chat, real-time), separate WebSocket workers from HTTP workers."

**Q10: "What's the lifecycle of a FastAPI request?"**

> "(1) Uvicorn receives raw HTTP, parses to ASGI scope. (2) Outermost middleware runs — last registered = outermost. (3) Routing matches path to operation. (4) Dependencies are resolved in graph order, with caching per request. (5) Pydantic validates the body and parameters. (6) Endpoint executes. If `async def` → on event loop; if `def` → in threadpool. (7) Return value passes through `response_model` filter. (8) Pydantic serializes to JSON. (9) Middleware unwinds in reverse order. (10) Uvicorn writes response. Knowing this lifecycle is crucial for placing custom logic at the right layer."

---

### 51. 📌 The Master TLDR for FastAPI

> "FastAPI is Starlette (ASGI) + Pydantic (validation) + Dependency Injection + auto OpenAPI. Tech-lead-level mastery is about: (1) **Async vs sync endpoint discipline** — never put sync I/O inside `async def` (blocks event loop). (2) **Dependency injection as the central design pattern** — for auth, DB sessions, services, with test overrides via `app.dependency_overrides`. (3) **Pydantic v2 with separate input/output models** — `response_model` filters out fields like password. (4) **Domain-driven project structure** — `users/router.py` + `users/service.py` (framework-agnostic) + `users/repository.py`. (5) **Use `lifespan`, not deprecated startup/shutdown events.** (6) **For critical async work, Celery + Outbox pattern, never `BackgroundTasks`.** (7) **Production**: Gunicorn + UvicornWorker, `2×CPU+1` workers, Nginx in front, PgBouncer for Postgres, Redis for caching, Prometheus + OpenTelemetry for observability, disable `/docs` in prod."

---

### 52. Migration Management — Alembic Deep Dive & Versioning

> **📣 Definition:** _"FastAPI has no built-in migration system — Alembic (by the SQLAlchemy author) is the standard. Alembic tracks schema changes as a linear chain of version files, each with `upgrade()` and `downgrade()` functions. Key production concerns: autogenerate catches model changes but misses renames and data migrations; always review generated files. Use branch labels for parallel development, stamp for baseline migrations, and never auto-apply in production without review."_

```python
# === Initial Setup ===
# pip install alembic
# alembic init -t async migrations       # async template for async SQLAlchemy

# migrations/env.py — configure to read your models
from app.database import Base
from app.models import *                  # import ALL models so Alembic sees them

target_metadata = Base.metadata           # Alembic compares this against the DB


# === Core Commands ===
alembic revision --autogenerate -m "add users table"    # generate from model diff
alembic upgrade head                                     # apply all pending
alembic upgrade +1                                       # apply next one only
alembic downgrade -1                                     # rollback one
alembic downgrade base                                   # rollback ALL
alembic current                                          # show current version
alembic history --verbose                                # show migration chain
alembic stamp head                                       # mark DB as up-to-date (no-op apply)


# === Migration File Anatomy ===
"""add users table

Revision ID: a1b2c3d4e5f6
Revises: (previous revision)
Create Date: 2025-01-15 10:30:00
"""
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        "users",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("email", sa.String(255), unique=True, nullable=False),
        sa.Column("created_at", sa.DateTime, server_default=sa.func.now()),
    )
    op.create_index("ix_users_email", "users", ["email"])

def downgrade():
    op.drop_index("ix_users_email")
    op.drop_table("users")


# === Data Migration (manual revision) ===
alembic revision -m "backfill user roles"     # no --autogenerate!

def upgrade():
    # Use raw SQL or op.execute() for data migrations
    op.execute("UPDATE users SET role = 'member' WHERE role IS NULL")

def downgrade():
    op.execute("UPDATE users SET role = NULL WHERE role = 'member'")


# === What Autogenerate CATCHES vs MISSES ===
# ✅ Catches: new tables, new columns, dropped columns, type changes, index changes
# ❌ Misses:  column renames (sees drop+add), data migrations, constraint names,
#             table renames, enum value changes, custom types
# ALWAYS review autogenerated files before applying!


# === Handling Migration Conflicts in Teams ===
# Two devs create migrations from the same base → branched history
alembic merge heads -m "merge feature_a and feature_b"
# Creates a merge point that depends on both branches

# Prevention: use a CI check that fails if there are multiple heads
alembic heads         # should always return exactly 1 head
```

**Alembic versioning & environment management:**

| Scenario                       | Command                                     | Notes                                         |
| ------------------------------ | ------------------------------------------- | --------------------------------------------- |
| First migration on existing DB | `alembic stamp head`                        | Mark current state without running migrations |
| Check pending migrations       | `alembic current` vs `alembic heads`        | If different → pending migrations             |
| CI/CD check                    | `alembic check` (Alembic 1.9+)              | Fails if models differ from latest migration  |
| Rollback in production         | `alembic downgrade -1`                      | Only works if `downgrade()` is implemented    |
| Skip a bad migration           | Edit the migration chain or `stamp` past it | Emergency only — document thoroughly          |

📌 **TLDR:** "Alembic is the SQLAlchemy migration tool. `--autogenerate` detects model changes but misses renames and data migrations — always review. Use `stamp head` for existing databases. Use `merge heads` for branch conflicts. Always implement `downgrade()`. Run `alembic check` in CI. Never auto-apply to production without review."

---

### 53. Pydantic v2 — Deep Dive for Production

> **📣 Definition:** _"Pydantic v2 (rewritten in Rust via pydantic-core) is 5-50× faster than v1. Beyond basic validation, production usage requires: model inheritance, computed fields, custom validators with `@field_validator` and `@model_validator`, discriminated unions for polymorphic payloads, `model_config` for strict/lax mode, serialization control with `model_dump(exclude=...)`, and `pydantic-settings` for environment-driven configuration. The core pattern: separate input schemas (`Create`), output schemas (`Response`), and update schemas (`Patch`) — never reuse one model for all."_

```python
from pydantic import BaseModel, Field, field_validator, model_validator, ConfigDict
from datetime import datetime
from enum import Enum


# === Separate Input / Output / Update Models ===
class UserCreate(BaseModel):
    """Input: what the client sends."""
    email: str
    password: str = Field(min_length=8)
    full_name: str = Field(min_length=2, max_length=100)

class UserUpdate(BaseModel):
    """Partial update: all fields optional."""
    email: str | None = None
    full_name: str | None = None

class UserResponse(BaseModel):
    """Output: what the client sees. NO password."""
    model_config = ConfigDict(from_attributes=True)    # read from ORM objects

    id: int
    email: str
    full_name: str
    created_at: datetime
    is_active: bool


# === Validators ===
class OrderCreate(BaseModel):
    product_id: int
    quantity: int = Field(gt=0, le=1000)
    discount_code: str | None = None

    @field_validator("discount_code")
    @classmethod
    def normalize_discount(cls, v):
        return v.upper().strip() if v else v

    @model_validator(mode="after")
    def check_bulk_discount(self):
        if self.quantity > 100 and not self.discount_code:
            raise ValueError("Bulk orders (>100) require a discount code")
        return self


# === Discriminated Unions (polymorphic payloads) ===
from typing import Literal, Annotated
from pydantic import Discriminator

class CreditCardPayment(BaseModel):
    type: Literal["credit_card"]
    card_number: str
    expiry: str

class UPIPayment(BaseModel):
    type: Literal["upi"]
    vpa: str

PaymentMethod = Annotated[
    CreditCardPayment | UPIPayment,
    Discriminator("type"),            # Pydantic picks the right model based on "type" field
]

class CheckoutRequest(BaseModel):
    order_id: int
    payment: PaymentMethod            # {"type": "upi", "vpa": "user@bank"}


# === Computed Fields (v2) ===
from pydantic import computed_field

class Product(BaseModel):
    price: float
    tax_rate: float = 0.18

    @computed_field
    @property
    def total_price(self) -> float:
        return round(self.price * (1 + self.tax_rate), 2)


# === pydantic-settings (environment-driven config) ===
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = ConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
    )

    database_url: str                                  # DATABASE_URL in .env
    redis_url: str = "redis://localhost:6379/0"
    secret_key: str
    debug: bool = False
    allowed_origins: list[str] = ["http://localhost:3000"]

settings = Settings()    # auto-reads from env vars / .env file


# === Serialization Control ===
user = UserResponse(id=1, email="t@x.com", full_name="Tushar", ...)

user.model_dump()                                    # → dict (all fields)
user.model_dump(exclude={"created_at"})              # → dict without created_at
user.model_dump(include={"id", "email"})             # → only id + email
user.model_dump(exclude_none=True)                   # → skip None values
user.model_dump_json()                               # → JSON string directly
```

**v1 → v2 migration cheat sheet:**

| v1                | v2                               | Notes                   |
| ----------------- | -------------------------------- | ----------------------- |
| `class Config:`   | `model_config = ConfigDict(...)` | Class-level config dict |
| `orm_mode = True` | `from_attributes = True`         | Read from ORM objects   |
| `@validator`      | `@field_validator`               | Must be `@classmethod`  |
| `@root_validator` | `@model_validator(mode="after")` | Or `mode="before"`      |
| `.dict()`         | `.model_dump()`                  | Renamed                 |
| `.json()`         | `.model_dump_json()`             | Renamed                 |
| `schema_json()`   | `model_json_schema()`            | Renamed                 |
| `Optional[str]`   | `str \| None = None`             | Python 3.10+ syntax     |

📌 **TLDR:** "Pydantic v2 is Rust-powered (5-50× faster). Separate `Create`/`Update`/`Response` models — never reuse. Use `@field_validator` for field-level, `@model_validator` for cross-field. `ConfigDict(from_attributes=True)` for ORM. Discriminated unions for polymorphic payloads. `pydantic-settings` for env config. `model_dump(exclude=...)` for serialization control."

---

### 54. Migration Alternatives & Advanced Strategies

> **📣 Definition:** _"Alembic is the standard for SQLAlchemy projects, but there are alternatives and complementary tools: `yoyo-migrations` (framework-agnostic, raw SQL), `migra` (auto-generates diffs between two databases), and `sqlalchemy-utils` for common schema patterns. The best approach is often Alembic + complementary tooling: use `migra` to verify Alembic didn't miss anything, wrap migrations in CI checks, and use Alembic's `--sql` mode for DBA-reviewed changes in regulated environments."_

```python
# === Alternative 1: yoyo-migrations (raw SQL, framework-agnostic) ===
# pip install yoyo-migrations
# Works without SQLAlchemy — just raw SQL files

# migrations/0001_create_users.py
from yoyo import step

steps = [
    step(
        "CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(255) UNIQUE)",
        "DROP TABLE users",     # rollback
    ),
    step(
        "CREATE INDEX ix_users_email ON users (email)",
        "DROP INDEX ix_users_email",
    ),
]

# Apply:  yoyo apply --database postgresql://localhost/mydb ./migrations
# Rollback: yoyo rollback --database postgresql://localhost/mydb ./migrations

# When to use yoyo:
# - Non-SQLAlchemy projects (Tortoise ORM, raw asyncpg, encode/databases)
# - Teams that prefer raw SQL over Python migration code
# - Polyglot environments (same migration tool for Python + Go services)


# === Alternative 2: migra (schema diff tool) ===
# pip install migra
# Compares two databases and generates the ALTER statements to sync them

# Usage: migra postgresql://localhost/db_old postgresql://localhost/db_new
# Output: ALTER TABLE, CREATE INDEX, etc. — the exact SQL to sync schemas

# Best use: verification layer on top of Alembic
# Run migra after applying Alembic migrations to check if any changes were missed


# === Alembic Advanced: Offline / SQL Mode (DBA-reviewed) ===
alembic upgrade head --sql > migration.sql
# Generates raw SQL file without executing
# DBA reviews and applies manually — required in regulated environments (finance, healthcare)


# === Alembic + CI/CD Best Practices ===
# 1. In CI: verify no pending model changes
alembic check                        # fails if models differ from latest migration

# 2. In CI: verify migration chain integrity
alembic heads                        # must return exactly 1 head (no branches)

# 3. In CI: test migrations on a fresh DB
alembic upgrade head                 # apply all from scratch
alembic downgrade base               # rollback all
alembic upgrade head                 # re-apply — full round-trip test

# 4. In deployment pipeline:
alembic upgrade head --sql | psql    # apply via psql for better error handling
```

**Migration tool comparison:**

| Tool                  | Best For                      | ORM             | SQL Control         | Rollback         |
| --------------------- | ----------------------------- | --------------- | ------------------- | ---------------- |
| **Alembic**           | SQLAlchemy projects           | SQLAlchemy only | Via `--sql` mode    | `downgrade()`    |
| **yoyo**              | Raw SQL / any ORM             | Any / None      | Full SQL            | `step()` reverse |
| **migra**             | Schema diffing / verification | Any             | Generates SQL diffs | N/A (diff tool)  |
| **Django migrations** | Django projects               | Django ORM      | `sqlmigrate`        | Auto-generated   |

📌 **TLDR:** "Alembic is the standard for SQLAlchemy but not the only option. `yoyo-migrations` for raw SQL / non-SQLAlchemy projects. `migra` for schema diff verification. Use Alembic's `--sql` mode for DBA-reviewed changes. In CI: `alembic check` for drift, `alembic heads` for branch conflicts, round-trip test (up → down → up). Wrap all of this in your deployment pipeline."

---

# Aggregation

## Basics

Aggregations in Django allow you to perform calculations across a **QuerySet** (a set of objects) rather than dealing with individual objects. They use database functions (like `SUM`, `AVG`, `MAX`, `MIN`, `COUNT`) to calculate values and return a single result. 📈

To use aggregations, you primarily rely on the **`aggregate()`** and **`annotate()`** methods, which require importing functions from `django.db.models`.

---

### 📊 1. The `aggregate()` Method (Single Result)

The `aggregate()` method is used to calculate a summary value for the **entire QuerySet**. It returns a **dictionary** where the key is the calculation's name and the value is the result.

#### Setup

Let's assume a model called `Sale`:

```python
from django.db import models

class Sale(models.Model):
    amount = models.DecimalField(max_digits=10, decimal_places=2)
    quantity = models.IntegerField()
    # ... other fields
```

#### Examples

| Goal        | Method  | Example Code                                                                  | Result (Dictionary)          |
| :---------- | :------ | :---------------------------------------------------------------------------- | :--------------------------- |
| **Average** | `Avg`   | `Sale.objects.aggregate(average_sale=Avg('amount'))`                          | `{'average_sale': 55.25}`    |
| **Maximum** | `Max`   | `Sale.objects.aggregate(highest_amount=Max('amount'))`                        | `{'highest_amount': 120.00}` |
| **Count**   | `Count` | `Sale.objects.filter(quantity__gte=5).aggregate(total_big_sales=Count('id'))` | `{'total_big_sales': 15}`    |

---

### 🏷️ 2. The `annotate()` Method (Value Per Item)

The `annotate()` method adds a calculated field to **every object** in the QuerySet. It's often used with `Count` for grouping, similar to a `GROUP BY` clause in SQL.

#### Example: Counting Related Objects

Assume a **`Book`** model has a `ForeignKey` to an **`Author`** model.

**Goal:** Get a list of all authors and the **number of books** each has written.

```python
from django.db.models import Count

# 'book' is the reverse relationship name from Author to Book
authors = Author.objects.annotate(
    num_books=Count('book')
)
```

**Result Usage:** You can loop through the `authors` QuerySet, and each author instance will now have a `num_books` attribute:

```python
for author in authors:
    # Access the calculated field
    print(f"{author.name} has written {author.num_books} books.")
```

#### Example: Combining with Filtering

You can filter the results based on the annotated field.

**Goal:** Find only authors who have written **more than 5** books.

```python
big_authors = Author.objects.annotate(
    num_books=Count('book')
).filter(
    num_books__gt=5 # Filter based on the calculated 'num_books' field
)
```

---

## Conditional Aggregation

Using a **filter inside `Count()` or other aggregate functions** allows you to count only the _specific related items_ that match a condition, rather than counting all of them.

This technique is called **Conditional Aggregation**.

---

### Conditional Aggregation with `Count`

When you use the standard `Count('related_field')` inside `annotate()`, Django counts **all** related objects for each item in your QuerySet.

To count only a subset of related objects, you pass a **`Q` object** to the aggregate function.

#### Setup Example

Let's assume we have two models:

- **`Author`**
- **`Book`** (with a ForeignKey to `Author` and a `status` field: 'published' or 'draft')

#### 1\. Simple Count (Counting All)

This counts _all_ books for each author, regardless of status:

```python
from django.db.models import Count

Author.objects.annotate(
    total_books=Count('book')
)
# Result: author.total_books includes published AND draft books.
```

---

### 2\. Filtering the Count (Conditional Aggregation)

To count only the published books, you use a `Q` object **inside** the `Count` function.

#### A. Counting Specific Subsets

**Goal:** Get a list of authors and the count of their **published books only**.

```python
from django.db.models import Count, Q

authors = Author.objects.annotate(
    # Use a Q object to filter the relationship *before* counting
    published_count=Count('book', filter=Q(book__status='published'))
)
# Result: author.published_count only includes books where status is 'published'.
```

**How it works:** The `filter` argument inside `Count()` restricts which objects from the `book` relationship are included in the count for that author.

#### B. Counting Different Subsets

You can add multiple conditional counts in the same QuerySet:

**Goal:** Count published books AND draft books separately.

```python
authors = Author.objects.annotate(
    published_count=Count('book', filter=Q(book__status='published')),
    draft_count=Count('book', filter=Q(book__status='draft'))
)
# Result: QuerySet now has both author.published_count and author.draft_count.
```

### 🧠 Application to Other Aggregates

This technique works with all aggregation functions, not just `Count`:

| Function  | Goal                                                                         | Example                                                   |
| :-------- | :--------------------------------------------------------------------------- | :-------------------------------------------------------- |
| **`Avg`** | Find the **average** rating of only the **published** books for each author. | `Avg('book__rating', filter=Q(book__status='published'))` |
| **`Sum`** | Find the **total quantity** of books sold only in the **last month**.        | `Sum('sale__quantity', filter=Q(sale__date__month=10))`   |

**Key Takeaway:** When you need to summarize a related dataset but only want to include records that meet a specific condition, use the **`filter=Q(...)`** argument **inside** your aggregate function. This is often simpler and more efficient than using the complex `Case` and `When` expressions sometimes required in older Django versions.

---

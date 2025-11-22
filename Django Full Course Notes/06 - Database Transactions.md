# Database Transactions

Database transactions are fundamental concepts that ensure data integrity and reliability in a database system. A **transaction** is a sequence of one or more SQL operations treated as a single, indivisible unit of work.

---

## Atomic Transactions (The ACID Principle)

The core concept that defines a transaction is the **ACID** properties. **Atomicity** is the "A" in ACID, and it's the most crucial aspect of a transaction.

### Atomicity (All or Nothing)

Atomicity guarantees that all changes made within a transaction are treated as a single, atomic operation.

- If **all** operations within the transaction succeed, the transaction is **committed**, and all changes are permanently recorded in the database.
- If **any** single operation within the transaction fails (due to an error, constraint violation, system crash, etc.), the transaction is **rolled back**. All changes made up to that point are undone, and the database is returned to its state before the transaction began.

**Analogy:** Think of a bank transfer. The transaction involves two steps: **debiting** the sender's account and **crediting** the recipient's account. Atomicity ensures that either **both** steps happen, or **neither** happens. If the debit succeeds but the credit fails, the entire operation is rolled back to prevent money from disappearing.

---

## The Other ACID Properties

For a system to be considered truly transactional, it must also satisfy the other three ACID properties:

| Property        | Meaning                                                                                                               | Ensures                                                                                                          |
| :-------------- | :-------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------- |
| **C**onsistency | A transaction moves the database from one valid state to another.                                                     | Prevents illegal transactions (e.g., trying to set a required field to null).                                    |
| **I**solation   | Concurrent transactions execute without interfering with one another.                                                 | Ensures that a user reading data doesn't see partial, uncommitted results from another simultaneous transaction. |
| **D**urability  | Once a transaction is committed, its changes are permanent, even in the event of a system failure (e.g., power loss). | Guarantees that committed data is securely written to non-volatile storage.                                      |

---

## Transactions in Django

Django manages database connections and transactions by default. In most views, Django operates in **autocommit mode**, meaning every ORM action is a separate transaction.

However, for complex operations, you explicitly manage transactions using decorators or context managers:

### 1\. `atomic()` Context Manager (Recommended)

This is the standard and cleanest way to ensure a block of code executes atomically.

```python
from django.db import transaction

# If any code inside this 'with' block fails, all database changes are rolled back.
with transaction.atomic():
    # 1. Update the inventory (Step 1 of the transaction)
    product = Product.objects.select_for_update().get(pk=product_id)
    product.inventory -= 1
    product.save()

    # 2. Create the order record (Step 2 of the transaction)
    Order.objects.create(...)

    # If a crash happens here, both the inventory change and the new order are reversed.
# Transaction is committed here if no exceptions occurred.
```

### 2\. Preventing Race Conditions

The `transaction.atomic()` manager is often paired with `select_for_update()` (as shown above) to ensure that the row being modified (the `product` inventory) is **locked** by the current transaction until it commits. This prevents a **race condition** where two users simultaneously try to buy the last item.

---

Decorators and `select_for_update()` are the two key tools for managing concurrency and complex transactions in Django.

---

## Transaction Decorator: `@transaction.atomic`

The `@transaction.atomic` decorator is the most common way to enforce atomicity on an entire function or method. It wraps the function's logic in a `try...except` block, ensuring **all database operations are treated as a single, atomic unit.**

### How to Use the Decorator

You apply the decorator directly to the function definition, typically within a Django view or a standalone helper function:

```python
from django.db import transaction
# from django.shortcuts import render # if this were a view

@transaction.atomic
def process_full_order(user, cart_items):
    # Start of the atomic block

    # 1. Create the main Order header
    new_order = Order.objects.create(user=user, status='PENDING')

    # 2. Loop through and create line items
    for item in cart_items:
        # If this line fails (e.g., product_id is invalid),
        # the exception is raised, and the entire transaction is rolled back.
        OrderItem.objects.create(order=new_order, product=item.product, quantity=item.qty)

    # 3. Clear the user's cart
    Cart.objects.filter(user=user).delete()

    # End of the atomic block
    # If the function reaches here without error, the transaction is COMMITTED.
    # If any error occurred in steps 1, 2, or 3, all steps are ROLLED BACK.

    return new_order
```

### Decorator vs. Context Manager

The decorator is simply syntactic sugar for the context manager:

| Method              | Syntax                       | Usage Context                                                                            |
| :------------------ | :--------------------------- | :--------------------------------------------------------------------------------------- |
| **Decorator**       | `@transaction.atomic`        | Best for enforcing atomicity over an **entire function/method**.                         |
| **Context Manager** | `with transaction.atomic():` | Best for enforcing atomicity over a **specific block of code** within a larger function. |

---

### The `select_for_update()` QuerySet Method

While transactions ensure that data is committed atomically, they don't automatically prevent a **race condition**—when two users try to update the same piece of data at the same time. This is where **`select_for_update()`** comes in.

#### Purpose: Pessimistic Locking

`select_for_update()` performs **pessimistic locking** at the database level. When you call it on a QuerySet, the database places a lock on the rows returned, and this lock is held **until the current transaction is committed or rolled back.**

Any other transaction attempting to read or modify those locked rows will be forced to wait until the lock is released.

#### Example: Preventing Double-Sale

This is critical for managing inventory or balances:

```python
from django.db import transaction
# (Assuming Product has an 'inventory' field)

def sell_item(product_id, quantity_to_buy):
    try:
        with transaction.atomic():
            # 1. Acquire Lock: The database locks the row for this specific product ID.
            # No other concurrent transaction can read or write to this row until the 'with' block ends.
            product = Product.objects.select_for_update().get(pk=product_id)

            if product.inventory < quantity_to_buy:
                # Raise an exception, triggering a rollback
                raise ValueError("Out of stock")

            # 2. Update Data
            product.inventory -= quantity_to_buy
            product.save()

            # ... other steps (e.g., creating an order record)

        # Lock is released and changes are committed here
        return True

    except ValueError:
        # Transaction is implicitly rolled back
        return False
```

#### Key Rules for `select_for_update()`

1.  **Must be in an Atomic Transaction:** `select_for_update()` has no effect outside of a `transaction.atomic()` block.
2.  **Locks the Database Row:** It requires the database to support row-level locking (which PostgreSQL and MySQL do).
3.  **Prevents Concurrent Modification:** It forces concurrent operations to wait their turn, guaranteeing that your logic (like checking inventory quantity) is performed based on the absolute, current state of the database.

---

### `on_commit()`

The `on_commit()` method in Django transactions allows you to schedule a function to be executed **only after** the current atomic transaction successfully **commits** to the database.

This is a crucial tool for avoiding data inconsistencies when dealing with operations that occur _outside_ the database, such as sending emails, dispatching background tasks, or performing I/O.

---

#### Why `on_commit()` is Necessary

When you run code inside a `transaction.atomic()` block, any exceptions will cause the transaction to roll back, but Django's Python code will still execute sequentially.

| Situation                  | Standard Python Code                     | Code Inside `on_commit()`                     |
| :------------------------- | :--------------------------------------- | :-------------------------------------------- |
| **Commit Succeeds**        | Runs immediately.                        | Runs **after** the commit. (Desired behavior) |
| **Transaction Rolls Back** | Runs immediately, _before_ the rollback. | **Does not run.** (Desired behavior)          |

If you send a welcome email inside an `atomic` block and the subsequent database saving fails, the transaction rolls back (the new user is deleted), but the **email has already been sent**. `on_commit()` prevents this.

---

### How to Use `on_commit()`

You use `on_commit()` inside an `atomic` block or any function decorated with `@transaction.atomic`.

#### 1\. Simple Usage

You pass the function you want to execute as the argument to `on_commit()`.

```python
from django.db import transaction

def send_welcome_email(user_id):
    print(f"DEBUG: Sending welcome email to User ID {user_id}...")
    # This function would typically dispatch a task to a worker queue (like Celery)

def create_user_and_notify(username, email):
    with transaction.atomic():
        # 1. Database operation: Create the new user
        user = User.objects.create(username=username, email=email)

        # 2. Schedule non-DB action: Send email ONLY if the user save succeeds
        transaction.on_commit(lambda: send_welcome_email(user.id))

        # 3. Simulate a failure (to show rollback)
        # 1 / 0

    # If a failure occurs in step 3:
    # - The user is rolled back (never saved).
    # - The send_welcome_email function is NEVER executed.
```

#### 2\. Passing Arguments

Since `on_commit` requires a callable with no arguments (or a lambda), you often use a `lambda` function to pass arguments from the transaction context (like `user.id`) to your scheduled function.

---

### Use Cases for `on_commit()`

`on_commit()` is essential for coordinating actions that depend on the database being in a guaranteed state:

- **Background Tasks:** Dispatching jobs (e.g., image resizing, report generation) via Celery or other task queues.
- **External Notifications:** Sending emails, webhooks, or push notifications.
- **Cache Invalidation:** Clearing specific cache keys that are dependent on the committed data.

---

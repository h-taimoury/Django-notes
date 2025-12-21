# Django notes

## Question:

Tell me again, why the `choices` argument in a `CharField` should be like this?

```python
STATUS_CHOICES = (
        ("pending", "Pending"),
        ("paid", "Paid"),
        ("shipped", "Shipped"),
        ("delivered", "Delivered"),
        ("cancelled", "Cancelled"),
    )
```

In `("pending", "Pending")`, as an example, is the first string the one that actually will be stored in the database and the second one is just a label? Where this label will be used?

## Answer:

You are exactly right. The structure of the tuple is defined as `(value_to_store, human_readable_label)`.

### 1. The First String: `"pending"` (The Database Value)

- **What it is:** This is the "Data." It is exactly what is written into the SQL column in your database.
- **Why use lowercase?** It is standard practice to use lowercase for the database value because it makes queries cleaner and more consistent across different database systems (PostgreSQL, MySQL, etc.).

### 2. The Second String: `"Pending"` (The Label)

- **What it is:** This is the "Display." It is a human-friendly version of the data.
- **Where is it used?**
- **Django Admin:** When you look at an Order in the admin panel, you will see a dropdown menu. The admin uses the **Label** in the dropdown so the administrator sees "Pending" instead of the raw code "pending".
- **Forms & Serializers:** If you use this model in a `ModelForm` or if you auto-generate a Serializer, the labels are used as the display text for the choices.
- **In your code:** Django provides a "magic" method for every field with choices called `get_FOO_display()`.

> **Example:**
> If you have an order object: `order = Order.objects.get(id=1)`
>
> - `order.status` returns `"pending"`
> - `order.get_status_display()` returns `"Pending"`

### Why use this two-part system?

This separation is very powerful because it allows you to change the **Label** without migrating your database.

Imagine your boss says, "I don't like the word 'Paid', let's call it 'Success' instead."

- **Without Choices:** You would have to run a database migration to update every single row from "paid" to "success".
- **With Choices:** You simply change the label in your tuple to `("paid", "Success")`. The database still stores "paid," but every user on the website and every admin in the panel sees "Success."

---

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

## Question:

## Answer:

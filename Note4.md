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

In `AddressViewSet` in our E-Commerce project, this is the `get_queryset` method:

```python
def get_queryset(self):
    # Users can only see and manage their own addresses
    return Address.objects.filter(user=self.request.user)
```

A problem that comes to my mind is that for listing the user's addresses, this queryset is great, but when creating the first address, this queryset `Address.objects.filter(user=self.request.user)` returns nothing, right? So doesn't it cause a problem?

## Answer:

That is a very insightful question! It’s a common point of confusion when first working with Django Rest Framework's `ViewSet` logic.

The short answer is: **No, it will not be a problem.**

Here is the technical reason why `get_queryset` returning an empty list doesn't break the **Create** action:

### 1. The difference between "List" and "Create" logic

In a DRF `ModelViewSet`, the internal logic treats these two actions very differently:

- **List/Retrieve/Update/Delete:** These actions focus on **existing** data. They use the `queryset` to know which records are "available" to the user. If `get_queryset` returns nothing, `list` returns an empty array `[]`, and `retrieve` returns a `404 Not Found`.
- **Create:** This action does **not** look at the existing queryset to validate the new object. Instead, it uses the **Serializer** to validate the incoming data and then calls `serializer.save()`.

### 2. How `perform_create` works

When you call `POST` to create your first address, the sequence looks like this:

1. DRF initializes the Serializer with your data.
2. It runs `is_valid()`.
3. It calls `self.perform_create(serializer)`.
4. Inside your `perform_create`, you call `serializer.save(user=self.request.user)`.

The database then creates a **new row**. The fact that `get_queryset()` was empty a millisecond ago doesn't matter because `create` is an **insert** operation, not a **selection** operation.

### 3. Why `get_queryset` is still vital for security

Even though it doesn't affect creation, having `filter(user=self.request.user)` is your "Security Guard."

If a malicious user tries to **Update** (PUT) address ID `500` which belongs to _another_ user, your `get_queryset` will look for ID `500` within **only that user's addresses**. Since it won't find it, it returns a `404 Not Found` instead of accidentally letting them edit someone else's data.

### Summary

Your `get_queryset` is perfectly safe.

- **First Address:** `get_queryset` is empty POST works Database now has 1 record.
- **Second Address:** `get_queryset` now has 1 record POST works Database now has 2 records.

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

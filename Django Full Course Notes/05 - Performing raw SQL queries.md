# Performing raw SQL queries

Raw SQL queries are your escape hatch when the Django ORM is either too complex, too slow, or simply cannot express the required database operation. While the ORM handles most tasks, knowing how to drop down to raw SQL is essential for advanced development.

The primary method for running custom SQL is using Django's **`connection.cursor()`**.

---

## Executing Raw SQL with `connection.cursor()`

The `connection` object is Django's direct interface to the database connection. By using `cursor()`, you open a channel to execute SQL statements directly.

### 1\. The Basic Process

The standard procedure involves three steps:

1.  **Get the Connection:** Import the `connection` object.
2.  **Open the Cursor:** Create a database cursor from the connection.
3.  **Execute the SQL:** Use `cursor.execute()` to run your custom SQL.

<!-- end list -->

```python
from django.db import connection

def get_expensive_products(price_threshold):
    with connection.cursor() as cursor:
        # 1. Execute the query
        # Use placeholders (%s) for parameters to prevent SQL injection!
        cursor.execute(
            "SELECT name, price FROM products_product WHERE price > %s ORDER BY price DESC",
            [price_threshold]  # 2. Pass parameters as a list/tuple
        )

        # 3. Retrieve the results
        results = cursor.fetchall()

    return results
```

### 2\. Cursor Methods for Retrieving Data

Once you execute a `SELECT` statement, the cursor holds the results. You use these methods to fetch the data:

| Method                          | Description                                                                            |
| :------------------------------ | :------------------------------------------------------------------------------------- |
| **`cursor.fetchone()`**         | Returns the **next single row** as a tuple, or `None` when no more rows are available. |
| **`cursor.fetchmany(size=10)`** | Returns the **next set of rows** (default 10) as a list of tuples.                     |
| **`cursor.fetchall()`**         | Returns **all remaining rows** from the result set as a list of tuples.                |

---

## ⚠️ Important Considerations

### 1\. Preventing SQL Injection (Crucial\!)

**Never format SQL queries using f-strings or string concatenation** when including user-provided data. This is the primary cause of SQL injection security vulnerabilities.

| Unsafe (Vulnerable)                                                                                                                                                                                          | Safe (Secure)                                                    |
| :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------- |
| `cursor.execute(f"SELECT * FROM table WHERE id = {user_id}")`                                                                                                                                                | `cursor.execute("SELECT * FROM table WHERE id = %s", [user_id])` |
| **Explanation:** Always use `%s` placeholders in the SQL string, and pass the corresponding values as the second argument to `cursor.execute()`. Django/the database driver handles the secure substitution. |

### 2\. Non-`SELECT` Queries (Writes)

For `INSERT`, `UPDATE`, or `DELETE` queries that modify data, you still use `cursor.execute()`. The transaction management is typically handled automatically by Django's framework (unless you use `commit=False` in `settings.py` or manage transactions manually).

### 3\. Combining with the ORM

If you need a list of object IDs from a complex SQL query, you can fetch those IDs using raw SQL and then use the ORM to hydrate the final objects:

```python
with connection.cursor() as cursor:
    cursor.execute("SELECT id FROM products_product WHERE ...")
    id_tuples = cursor.fetchall()

# Convert list of tuples (e.g., [(1,), (5,), (10,)]) to a list of IDs
product_ids = [item[0] for item in id_tuples]

# Use the ORM to get the full Product objects
products = Product.objects.filter(id__in=product_ids)
```

---

## Using `objects.raw()`

The `objects.raw()` method is another way to perform raw SQL queries in Django, but its key distinction is that it executes a SQL query and returns **live model instances** (objects) instead of just plain data tuples.

The `raw()` method is called on a model's manager and expects a `SELECT` query as its first argument.

### 1\. The Basic Process

```python
# Assuming you have a Product model

# 1. Write the raw SQL query
raw_query = "SELECT id, name, price FROM products_product WHERE price > %s ORDER BY price DESC"

# 2. Execute the query using the model's manager
expensive_products = Product.objects.raw(raw_query, [100.00])

# 3. Iterate over the results
for p in expensive_products:
    print(f"Name: {p.name}, Price: {p.price}")
    # You can access fields using dot notation, just like regular model objects.
```

### 2\. Key Differences from `connection.cursor()`

| Feature         | `objects.raw()`                                                                                                                                        | `connection.cursor()`                                                                                                                     |
| :-------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| **Output**      | Returns a **`RawQuerySet`** of **live model instances**.                                                                                               | Returns raw data as a list of **tuples** (or dictionaries if configured).                                                                 |
| **Flexibility** | Must be a **`SELECT`** query and must include the model's **primary key** and any fields you use.                                                      | Can execute **any** SQL command (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `DROP`).                                                         |
| **Readability** | High, as it integrates seamlessly with the ORM.                                                                                                        | Lower, requires manual mapping of tuple data back to meaningful fields.                                                                   |
| **Use Case**    | Ideal for complex `SELECT` queries (e.g., using advanced PostgreSQL functions, complex JOINs) where you still need the output as Django model objects. | Ideal for **non-SELECT** queries, database maintenance, or when speed is paramount and you only need a few raw values (no model objects). |

In short, if you need to fetch data and want that data delivered as easy-to-use Django objects, **`objects.raw()` is the better choice.** If you are modifying the database structure or running non-`SELECT` queries, use **`connection.cursor()`**.

---

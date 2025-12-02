## Question:

Is it true to say the `ModelViewSet` inherits from all the model mixins we also use in DRF Generic Concrete views, so like in CreateAPIView that we could over-ride the `perform_create` method, in view sets we can also over-ride the `perform_create` method.
What I'm trying to say is that the methods we used to override in generics are the same as the methods we need to override in viewsets.

## Answer:

That is **absolutely true**! You have grasped one of the most powerful and fundamental concepts of Django Rest Framework architecture.

The methods you override in Generic Views are exactly the same ones you can override in a `ModelViewSet`.

This is because the power of the `ModelViewSet` doesn't come from being a single, massive class, but from inheriting a set of small, reusable classes called **Mixins**.

### The DRF Inheritance Chain

Your core assumption is correct: a `ModelViewSet` is essentially a collection of the same components that make up the concrete Generic Views.

#### 1. The Core Mixins

The methods you override are defined in the following **Mixins**:

| Mixin Class             | Provides Functionality for... | Key Hook Methods Available           |
| :---------------------- | :---------------------------- | :----------------------------------- |
| `ListModelMixin`        | `GET` (List)                  | `list()`                             |
| **`CreateModelMixin`**  | `POST` (Create)               | **`perform_create()`**, `create()`   |
| `RetrieveModelMixin`    | `GET` (Detail)                | `retrieve()`                         |
| **`UpdateModelMixin`**  | `PUT` (Update)                | **`perform_update()`**, `update()`   |
| **`DestroyModelMixin`** | `DELETE` (Destroy)            | **`perform_destroy()`**, `destroy()` |

#### 2. The View Composition

Here is how the classes are built from the ground up:

- **`CreateAPIView`** is composed of: `GenericAPIView` + `CreateModelMixin`.
  - It exposes `perform_create()`.
- **`RetrieveUpdateDestroyAPIView`** is composed of: `GenericAPIView` + `RetrieveModelMixin` + `UpdateModelMixin` + `DestroyModelMixin`.
  - It exposes `perform_update()`, `perform_destroy()`, etc.
- **`ModelViewSet`** is composed of: `GenericAPIView` + `ListModelMixin` + `RetrieveModelMixin` + `CreateModelMixin` + `UpdateModelMixin` + `DestroyModelMixin` + `ViewSetMixin`.
  - It exposes **all** of the above hooks!

### Conclusion

Because your `PostViewSet` is built from the same mixins that define the Generic Views, the key hook methods are identical:

| Generic View Method              | Purpose                         | Where to Find in ViewSet |
| :------------------------------- | :------------------------------ | :----------------------- |
| **`get_queryset()`**             | Base for all views              | `PostViewSet`            |
| **`get_serializer_class()`**     | Base for all views              | `PostViewSet`            |
| **`perform_create()`**           | Injection before POST save      | `PostViewSet`            |
| **`perform_update()`**           | Injection before PUT/PATCH save | `PostViewSet`            |
| **`perform_destroy()`**          | Logic before DELETE             | `PostViewSet`            |
| **`create()`, `update()`, etc.** | Customizing the final response  | `PostViewSet`            |

If you know how to customize a Generic View, you already know the exact methods needed to customize a `ModelViewSet`. The only difference is that the ViewSet puts all those hooks into one single class.

## Question:

Consider this custom `create` method in our `PostViewSet`:

```python
    def create(self, request, *args, **kwargs):
        """Overrides create to return the custom minimal success response."""
        response = super().create(
            request, *args, **kwargs
        )

        # Logic from the old FBV
        post_url = f"/posts/{self.created_instance.slug}-{self.created_instance.id}/"
        response.data = {
            "url": post_url,
            "message": "Post created successfully.",
        }
        return response
```

When calling `super().create(request, *args, **kwargs)`, does it know that we also have overridden the `perform_create` method? I'm asking this because we are calling the `create` method that exists on the parent where `perform_create` method is different from what we have over-ridden.

## Answer:

This is an outstanding question that demonstrates a deep understanding of the object-oriented nature of Django Rest Framework. The concept you are asking about is the core mechanism of **Polymorphism** and **Method Resolution Order (MRO)** in Python.

The answer is **Yes, it absolutely knows**. When `super().create()` is called, the default `create` method executes, and when it needs to call `perform_create`, it automatically executes _your_ overridden version.

Here is the explanation of why:

### 1\. The Power of `self` (Polymorphism)

The trick is that the parent's `create()` method **never hard-codes a call to its own `perform_create` method**. Instead, it uses the instance itself (`self`) to look up the method to call:

```python
# The default implementation in CreateModelMixin (The Parent Class)

def create(self, request, *args, **kwargs):
    # ... setup and validation code ...

    # 💥 The parent calls self.perform_create(serializer) 💥
    # It does NOT call CreateModelMixin.perform_create(self, serializer)
    self.perform_create(serializer)

    # ... response code ...
```

#### The Lookup Process

1.  Your code calls: `response = super().create(request, *args, **kwargs)`
2.  Python executes the parent's `create()` method.
3.  When the parent's code hits `self.perform_create(serializer)`:
    - `self` is still an instance of **your class**, the `PostViewSet`.
    - Python looks up the method starting at the most specific class (`PostViewSet`).
    - It finds your overridden `perform_create()` method, which contains the custom logic: `serializer.save(author=self.request.user)`.
    - It executes _your_ custom method.

This ensures that the parent class provides the **structure** (validation, serializer setup, response status), but the child class provides the specific **behavior** (how and what to save).

### 2\. The Role of `super()` in the Chain

The `super()` function is used only to safely call a method from a parent or sibling class _in the Method Resolution Order (MRO)_. It is a way of saying: "Execute the next version of this method in the inheritance chain."

In your case:

1.  When you call `super().create()`, you are explicitly jumping up to the `CreateModelMixin`'s version of `create()`.
2.  Once that parent method is running, the rest of the execution happens on the current instance (`self`), ensuring that all subsequent method calls land back in **your most specialized class** (`PostViewSet`).

This mechanism is the entire reason why Mixins and Generic Views work so elegantly—you only need to override the smallest possible "hook" (`perform_create`) to inject your specific logic, and the high-level methods (`create`, `update`) automatically pick up your customization.

# Django REST Framework Generics

## Default methods in `GenericAPIView`

Here is the source code of `GenericAPIView`:

```python
class GenericAPIView(views.APIView):
    """
    Base class for all other generic views.
    """
    # You'll need to either set these attributes,
    # or override `get_queryset()`/`get_serializer_class()`.
    # If you are overriding a view method, it is important that you call
    # `get_queryset()` instead of accessing the `queryset` property directly,
    # as `queryset` will get evaluated only once, and those results are cached
    # for all subsequent requests.
    queryset = None
    serializer_class = None

    # If you want to use object lookups other than pk, set 'lookup_field'.
    # For more complex lookup requirements override `get_object()`.
    lookup_field = 'pk'
    lookup_url_kwarg = None

    # The filter backend classes to use for queryset filtering
    filter_backends = api_settings.DEFAULT_FILTER_BACKENDS

    # The style to use for queryset pagination.
    pagination_class = api_settings.DEFAULT_PAGINATION_CLASS

    # Allow generic typing checking for generic views.
    def __class_getitem__(cls, *args, **kwargs):
        return cls

    def get_queryset(self):
        """
        Get the list of items for this view.
        This must be an iterable, and may be a queryset.
        Defaults to using `self.queryset`.

        This method should always be used rather than accessing `self.queryset`
        directly, as `self.queryset` gets evaluated only once, and those results
        are cached for all subsequent requests.

        You may want to override this if you need to provide different
        querysets depending on the incoming request.

        (Eg. return a list of items that is specific to the user)
        """
        assert self.queryset is not None, (
            "'%s' should either include a `queryset` attribute, "
            "or override the `get_queryset()` method."
            % self.__class__.__name__
        )

        queryset = self.queryset
        if isinstance(queryset, QuerySet):
            # Ensure queryset is re-evaluated on each request.
            queryset = queryset.all()
        return queryset

    def get_object(self):
        """
        Returns the object the view is displaying.

        You may want to override this if you need to provide non-standard
        queryset lookups.  Eg if objects are referenced using multiple
        keyword arguments in the url conf.
        """
        queryset = self.filter_queryset(self.get_queryset())

        # Perform the lookup filtering.
        lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

        assert lookup_url_kwarg in self.kwargs, (
            'Expected view %s to be called with a URL keyword argument '
            'named "%s". Fix your URL conf, or set the `.lookup_field` '
            'attribute on the view correctly.' %
            (self.__class__.__name__, lookup_url_kwarg)
        )

        filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
        obj = get_object_or_404(queryset, **filter_kwargs)

        # May raise a permission denied
        self.check_object_permissions(self.request, obj)

        return obj

    def get_serializer(self, *args, **kwargs):
        """
        Return the serializer instance that should be used for validating and
        deserializing input, and for serializing output.
        """
        serializer_class = self.get_serializer_class()
        kwargs.setdefault('context', self.get_serializer_context())
        return serializer_class(*args, **kwargs)

    def get_serializer_class(self):
        """
        Return the class to use for the serializer.
        Defaults to using `self.serializer_class`.

        You may want to override this if you need to provide different
        serializations depending on the incoming request.

        (Eg. admins get full serialization, others get basic serialization)
        """
        assert self.serializer_class is not None, (
            "'%s' should either include a `serializer_class` attribute, "
            "or override the `get_serializer_class()` method."
            % self.__class__.__name__
        )

        return self.serializer_class

    def get_serializer_context(self):
        """
        Extra context provided to the serializer class.
        """
        return {
            'request': self.request,
            'format': self.format_kwarg,
            'view': self
        }

    def filter_queryset(self, queryset):
        """
        Given a queryset, filter it with whichever filter backend is in use.

        You are unlikely to want to override this method, although you may need
        to call it either from a list view, or from a custom `get_object`
        method if you want to apply the configured filtering backend to the
        default queryset.
        """
        for backend in list(self.filter_backends):
            queryset = backend().filter_queryset(self.request, queryset, self)
        return queryset

    @property
    def paginator(self):
        """
        The paginator instance associated with the view, or `None`.
        """
        if not hasattr(self, '_paginator'):
            if self.pagination_class is None:
                self._paginator = None
            else:
                self._paginator = self.pagination_class()
        return self._paginator

    def paginate_queryset(self, queryset):
        """
        Return a single page of results, or `None` if pagination is disabled.
        """
        if self.paginator is None:
            return None
        return self.paginator.paginate_queryset(queryset, self.request, view=self)

    def get_paginated_response(self, data):
        """
        Return a paginated style `Response` object for the given output data.
        """
        assert self.paginator is not None
        return self.paginator.get_paginated_response(data)
```

### `get_queryset(self)`

Returns the queryset that should be used for list views, and that should be used as the base for lookups in detail views. Defaults to returning the queryset specified by the `queryset` attribute.

This method should always be used rather than accessing `self.queryset` directly, as `self.queryset` gets evaluated only once, and those results are cached for all subsequent requests.

May be overridden to provide dynamic behavior, such as returning a queryset, that is specific to the user making the request.

#### Definition:

```python
def get_queryset(self):
    """
    Get the list of items for this view.
    This must be an iterable, and may be a queryset.
    Defaults to using `self.queryset`.

    This method should always be used rather than accessing `self.queryset`
    directly, as `self.queryset` gets evaluated only once, and those results
    are cached for all subsequent requests.

    You may want to override this if you need to provide different
    querysets depending on the incoming request.

    (Eg. return a list of items that is specific to the user)
    """
    assert self.queryset is not None, (
        "'%s' should either include a `queryset` attribute, "
        "or override the `get_queryset()` method."
        % self.__class__.__name__
    )

    queryset = self.queryset
    if isinstance(queryset, QuerySet):
        # Ensure queryset is re-evaluated on each request.
        queryset = queryset.all()
    return queryset
```

#### Over-ride example (from documentation):

```python
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```

#### Summary notes to remember:

1. We need to set a `queryset` attribute `OR` override the `get_queryset()` method. if we do none of them, we get an error (the definition of `get_queryset()` method makes sure we get an error)
2. Always access the collection of objects for the view by calling `self.get_queryset()`.
   Never access the static attribute `self.queryset` directly within instance methods (like `filter_queryset` or a custom `get_object` implementation).
3. When you over-ride the `get_queryset()` method, the caching problem goes away, because if you have over-ridden the `get_queryset()` method, you haven't set a `queryset` attribute, so logically you `WILL` only be using `get_queryset()` method and get a fresh queryset each time. So the caching problem is gone.

### `get_object(self)`

Returns an object instance that should be used for detail views. Defaults to using the lookup_field parameter to filter the base queryset.

#### Definition:

```python
def get_object(self):
    """
    Returns the object the view is displaying.

    You may want to override this if you need to provide non-standard
    queryset lookups.  Eg if objects are referenced using multiple
    keyword arguments in the url conf.
    """
    queryset = self.filter_queryset(self.get_queryset())

    # Perform the lookup filtering.
    lookup_url_kwarg = self.lookup_url_kwarg or self.lookup_field

    assert lookup_url_kwarg in self.kwargs, (
        'Expected view %s to be called with a URL keyword argument '
        'named "%s". Fix your URL conf, or set the `.lookup_field` '
        'attribute on the view correctly.' %
        (self.__class__.__name__, lookup_url_kwarg)
    )

    filter_kwargs = {self.lookup_field: self.kwargs[lookup_url_kwarg]}
    obj = get_object_or_404(queryset, **filter_kwargs)

    # May raise a permission denied
    self.check_object_permissions(self.request, obj)

    return obj
```

#### Over-ride example (from documentation):

May be overridden to provide more complex behavior, such as object lookups based on more than one URL kwarg.

For example:

```python
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```

Note that if your API doesn't include any object level permissions, you may optionally exclude the `self.check_object_permissions`, and simply return the object from the `get_object_or_404` lookup.

---

---

---

---

## Save and deletion hooks:

The following methods are provided by the mixin classes, and provide easy overriding of the object save or deletion behavior.

### `perform_create(self, serializer)`

Called by `CreateModelMixin` when saving a new object instance.

#### Definition:

```python
def perform_create(self, serializer):
    serializer.save()
```

### `perform_update(self, serializer)`

Called by `UpdateModelMixin` when saving an existing object instance.

#### Definition:

```python
def perform_update(self, serializer):
    serializer.save()
```

### `perform_destroy(self, instance)`

Called by `DestroyModelMixin` when deleting an object instance.

#### Definition:

```python
def perform_destroy(self, instance):
    instance.delete()
```

These hooks are particularly useful for setting attributes that are implicit in the request, but are not part of the request data. For instance, you might set an attribute on the object based on the request user, or based on a URL keyword argument.

```python
def perform_create(self, serializer):
serializer.save(user=self.request.user)
```

These override points are also particularly useful for adding behavior that occurs before or after saving an object, such as emailing a confirmation, or logging the update.

```python
def perform_update(self, serializer):
instance = serializer.save()
send_email_confirmation(user=self.request.user, modified=instance)
```

You can also use these hooks to provide additional validation, by raising a ValidationError(). This can be useful if you need some validation logic to apply at the point of database save. For example:

```python
def perform_create(self, serializer):
queryset = SignupRequest.objects.filter(user=self.request.user)
if queryset.exists():
raise ValidationError('You have already signed up')
serializer.save(user=self.request.user)
```

### Let's make it clear

To truely understand why these hooks exist and how to use them, let's check the definition of `CreateModelMixin` as an example, specifically the `create` and `perform_create` methods:

```python
class CreateModelMixin:
    """
    Create a model instance.
    """
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()

    def get_success_headers(self, data):
        try:
            return {'Location': str(data[api_settings.URL_FIELD_NAME])}
        except (TypeError, KeyError):
            return {}
```

As you see, they didn't have to create this `perform_create` method because it only includes `serializer.save()`. They code simply put this line in the `create` method like this:

```python
def create(self, request, *args, **kwargs):
    serializer = self.get_serializer(data=request.data)
    serializer.is_valid(raise_exception=True)
    serializer.save() # Here they could simply run the save method on the serializer
    headers = self.get_success_headers(serializer.data)
    return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)
```

But why they didn't? Well, they provided the `perform_create` hook so we can inject custom logic at the time of creating an instance in the database.

To fully understand how we can customize this `perform_create` hook, we need to understand that:

1. To create a new instance, the `CreateModelMixin` calls the `create` method
2. The `perform_create` hook only contains one line: `serializer.save()`
3. In `create` method, the `perform_create` hook is called after these lines:

```python
serializer = self.get_serializer(data=request.data)
serializer.is_valid(raise_exception=True)
```

So it's called after the instantiation of the serializer and the validation check. So if you want to inject some logic at the level of serializer instantiation or validation, we can't achieve it by over-riding the `perform_create` method, but we have to over-ride the `create` method.

4. In `create` method, the `perform_create` hook is called before these lines:

```python
headers = self.get_success_headers(serializer.data)
return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)
```

So it's called before the headers is set and response is sent. So if you want to customize the headers or more importantly and commonly customize the response, we can't achieve that by over-riding the `perform_create` method, but we have to over-ride the `create` method.

I talked about `perform_create` hook and `CreateModelMixin` to teach you how we can customize the behaviour of `DRF Generics`. The same approach can be used for other two hooks and other model mixins too. I have provided the definition of all the model mixins for you in next section.

They created these three hooks because they thought the most common place to customize the `Generics` is where the logic inside these hooks happen, right? But obviously over-riding these hooks won't be enough for every customization need (specifically when we need to customize the response).

#### Note:

In every model mixin, there are one or two main methods and zero or some helper methods. For example in `CreateModelMixin`, the main method is `create` and the helper methods are `perform_create` and `get_success_headers`. What I want to emphasize is that the main method is what is called to do the main purpose of the mixin. For example the main purpose of the `CreateModelMixin` is to create a new resource and for that, the `create` method is called.

In `ListModelMixin`, there's only one method which is `list` and is the main method and is called when we need to list resources.

In `DestroyModelMixin`, the main method is `destroy` and the helper method is `perform_destroy`. When we need to delete a resource, the `destroy` method is called.

In `UpdateModelMixin`, there are two main methods: `update` and `partial_update`. For `PUT` requests, the `update` method is called and for `PATCH` requests, the `partial_update` method is called. Also there's one helper method which is `perform_update`.

---

---

## Mixins source code

### `CreateModelMixin`:

```python
class CreateModelMixin:
    """
    Create a model instance.
    """
    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        headers = self.get_success_headers(serializer.data)
        return Response(serializer.data, status=status.HTTP_201_CREATED, headers=headers)

    def perform_create(self, serializer):
        serializer.save()

    def get_success_headers(self, data):
        try:
            return {'Location': str(data[api_settings.URL_FIELD_NAME])}
        except (TypeError, KeyError):
            return {}
```

### `ListModelMixin`:

```python
class ListModelMixin:
    """
    List a queryset.
    """
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())

        page = self.paginate_queryset(queryset)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(queryset, many=True)
        return Response(serializer.data)
```

### `RetrieveModelMixin`:

```python
class RetrieveModelMixin:
    """
    Retrieve a model instance.
    """
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return Response(serializer.data)
```

### `UpdateModelMixin`:

```python
class UpdateModelMixin:
    """
    Update a model instance.
    """
    def update(self, request, *args, **kwargs):
        partial = kwargs.pop('partial', False)
        instance = self.get_object()
        serializer = self.get_serializer(instance, data=request.data, partial=partial)
        serializer.is_valid(raise_exception=True)
        self.perform_update(serializer)

        if getattr(instance, '_prefetched_objects_cache', None):
            # If 'prefetch_related' has been applied to a queryset, we need to
            # forcibly invalidate the prefetch cache on the instance.
            instance._prefetched_objects_cache = {}

        return Response(serializer.data)

    def perform_update(self, serializer):
        serializer.save()

    def partial_update(self, request, *args, **kwargs):
        kwargs['partial'] = True
        return self.update(request, *args, **kwargs)


```

### `DestroyModelMixin`:

```python
class DestroyModelMixin:
    """
    Destroy a model instance.
    """
    def destroy(self, request, *args, **kwargs):
        instance = self.get_object()
        self.perform_destroy(instance)
        return Response(status=status.HTTP_204_NO_CONTENT)

    def perform_destroy(self, instance):
        instance.delete()
```

## DRF Generics and ViewSets customization guidelines

1. If you need to customize the queryset (for example for admin users or based on actions in a ViewSet), override get_queryset method. You can see a good example in PostViewSet in our Blog_CBVs project.
2. If you need to customize the serializer (for example for using different serializers for different http requests), override get_serializer_class method. You can see a good example in PostViewSet in our Blog_CBVs project.
3. If you need to customize the create, update and partial_update methods when serializer.save() is called (for example to inject some implicit data in request like request.user which is not present in request.data), override perform_create and perform_update methods.
4. If you need to customize the destroy method when instance.delete() is called (for example to do something after the object is deleted), override perform_destroy method.
5. If you need to customize the response from create, update, partial_update and destroy methods, use these methods by calling them from super(), store the response then customize it like this:

```python
def create(self, request, *args, **kwargs):
    """Overrides create to return the custom minimal success response."""
    response = super().create(
        request, *args, **kwargs
    )
    post_url = f"/posts/{self.created_instance.slug}-{self.created_instance.id}/"
    response.data = {
        "url": post_url,
        "message": "Post created successfully.",
    }
    return response
```

### Note: Permission Classes

We know that the Generic Concrete Views (like CreateAPIView) inherit from GenericAPIView. The GenericAPIView inherits from APIView class itself. The reason I'm mentioning the APIView class is because permission_classes attribute and get_permissions method are not define in GenericAPIView but in APIView as following:

The permission_classes attribute:

```python
permission_classes = api_settings.DEFAULT_PERMISSION_CLASSES
```

The get_permissions method:

```python
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    return [permission() for permission in self.permission_classes]
```

We over-ride them like this:

The permission_classes attribute over-ridden:

```python
permission_classes = [IsAdminOrReadOnly]
```

Or

```python
permission_classes = [permissions.IsAuthenticated]
```

The get_permissions method over-ridden:

```python
def get_permissions(self):
    """
    Instantiates and returns the list of permissions that this view requires.
    """
    if self.action == 'list':
        permission_classes = [IsAuthenticated]
    else:
        permission_classes = [IsAdminUser]
    return [permission() for permission in permission_classes]
```

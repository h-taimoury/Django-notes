# Custom User Model

The recommended way to create a custom user model is to inherit from **`AbstractBaseUser`** and **`PermissionsMixin`**. This gives you full control over the authentication process and fields.

### 1\. Create the `users` App

Run:

```bash
py manage.py startapp users
```

### 2\. Define the custom `User` Model

Here is the code you need to put in new `users/models.py` file. Then I'll explain it:

```python
# users/models.py
from django.db import models
from django.contrib.auth.models import AbstractBaseUser, PermissionsMixin, BaseUserManager

# --- Custom User Manager ---
class UserManager(BaseUserManager):
    """
    Custom user model manager where email is the unique identifier
    for authentication instead of usernames.
    """
    def create_user(self, email, password, **extra_fields):
        """
        Creates and saves a User with the given email and password.
        """
        if not email:
            raise ValueError('The Email must be set')

        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password) # Handles password hashing
        user.save()
        return user

    def create_superuser(self, email, password, **extra_fields):
        """
        Creates and saves a Superuser with the given email and password.
        """
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)
        extra_fields.setdefault('is_active', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')

        return self.create_user(email, password, **extra_fields)

# --- Custom User Model ---
class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    first_name = models.CharField(max_length=30, blank=True)
    last_name = models.CharField(max_length=30, blank=True)
    date_joined = models.DateTimeField(auto_now_add=True)

    # Required fields for Django's permission system
    is_staff = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True) # Can be used for account verification later

    # Link the UserManager to the model
    objects = UserManager()

    # Specify the field used as the unique identifier for authentication
    USERNAME_FIELD = 'email'

    # Fields that are prompted for when creating a user via 'createsuperuser'
    REQUIRED_FIELDS = ['first_name', 'last_name']

    class Meta:
        verbose_name = 'user'
        verbose_name_plural = 'users'

    def __str__(self):
        return self.email

    def get_full_name(self):
        """
        Returns the first_name plus the last_name, with a space in between.
        """
        full_name = '%s %s' % (self.first_name, self.last_name)
        return full_name.strip()

    def get_short_name(self):
        """Returns the short name for the user."""
        return self.first_name
```

---

### 3\. Create a user serializer

Create a `serializers.py` file in new `users` app folder. Then put this code inside it:

```python
from rest_framework import serializers
from rest_framework_simplejwt.tokens import RefreshToken
from .models import User
from django.contrib.auth.hashers import make_password

# This is best practice for importing the User model (instead of the line above). It imports the user model dynamically using the AUTH_USER_MODEL setting.
# from django.contrib.auth import get_user_model
# User = get_user_model()

# class UserSerializer(serializers.ModelSerializer):
#     class Meta:
#         model = User
#         fields = "__all__"


class UserSerializer(serializers.ModelSerializer):
    # Add full_name field for easy API display (READ-ONLY)
    # This uses the model's get_full_name() method implicitly or explicitly via a method field.
    full_name = serializers.SerializerMethodField()

    # Define password explicitly as write-only for security and input control
    password = serializers.CharField(
        write_only=True,
        required=True,
        style={"input_type": "password"},
    )

    class Meta:
        model = User
        fields = (
            "id",
            "email",
            "first_name",
            "last_name",
            "full_name",
            "is_staff",
            "is_active",
            "created_at",
            "password",
        )
        # Add read_only_fields for fields that shouldn't be modifiable via standard input
        read_only_fields = ("id", "created_at")

    def get_full_name(self, obj):
        # Uses the method defined in your User model
        return obj.get_full_name()

    # Create method (for POST/Registration)
    def create(self, validated_data):
        # Explicitly pop the required arguments for clarity and to prevent ambiguity
        email = validated_data.pop("email")
        password = validated_data.pop("password")
        # Ensure password is included in validated_data for create_user to work
        user = User.objects.create_user(email, password, **validated_data)
        return user

    # Update method (for PUT/PATCH requests)

    # //* AI suggested this code which was not optimized because it made two database calls when password needed to be updated.
    # def update(self, instance, validated_data):
    #     """
    #     Custom update method to safely handle password hashing during PUT/PATCH.
    #     """
    #     # 1. Safely retrieve the password and remove it from the data
    #     password = validated_data.pop("password", None)

    #     # 2. Update all non-password fields (e.g., first_name, is_staff)
    #     instance = super().update(instance, validated_data)

    #     # 3. Hash and set the new password only if it was provided
    #     if password:
    #         instance.set_password(password)
    #         instance.save()

    #     return instance

    # Option 1:
    def update(self, instance, validated_data):
        # 1. Safely retrieve the password and remove it from the data
        raw_password = validated_data.pop("password", None)

        # 2. If a password was provided, hash it using make_password()
        #    and put the HASHED value back into validated_data.
        if raw_password:
            validated_data["password"] = make_password(raw_password)

        # 3. Call the parent's update method. It sees "password" in validated_data
        #    and correctly saves the HASHED string, along with all other fields,
        #    in a single database call.
        instance = super().update(instance, validated_data)

        return instance

    # Option 2:
    # def update(self, instance, validated_data):
    #     # 1. Safely retrieve the password and remove it from the data
    #     password = validated_data.pop("password", None)

    #     # 2. Hash the password if provided and assign it to the instance,
    #     #    but DO NOT save the instance yet.
    #     if password:
    #         instance.set_password(password)

    #     # 3. Use the parent's update method to apply all remaining fields
    #     #    (first_name, last_name, etc.) AND perform the SINGLE database save.
    #     instance = super().update(instance, validated_data)

    #     return instance


class UserSerializerWithToken(UserSerializer):
    token = serializers.SerializerMethodField(read_only=True)

    class Meta:
        model = User
        fields = ["id", "first_name", "last_name", "email", "is_staff", "token"]

    def get_token(self, obj):
        token = RefreshToken.for_user(obj)
        return str(token.access_token)

```

---

### 4\. Update `settings.py`

Crucially, before you run any migrations, you must tell Django to use your custom model by adding a setting in your project's `settings.py` file.

1.  Add `'users'` to your `INSTALLED_APPS`.
2.  Set `AUTH_USER_MODEL`.

<!-- end list -->

```python
# blog_project/settings.py

# ... other settings

INSTALLED_APPS = [
    # ... built-in apps
    'django.contrib.sites', # often required/helpful with custom auth
    # ...
    'users',  # <-- Add your app here
    'rest_framework', # You'll need this for DRF
    # ...
]

# ... other settings

# Specify your custom user model
AUTH_USER_MODEL = 'users.User' # <-- THIS IS THE KEY SETTING!
```

---

### 5\. Initial Migrations

Since this is the very first step, you can now safely create your initial migrations.

```bash
python manage.py makemigrations users
python manage.py migrate
```

You now have a fully functional custom user model using `email` for authentication.

---

## Explanations (as Questions and Answers):

Our custom user model inherits from AbstractBaseUser and PermissionsMixin. I'll provide to you with the source code of these two classes and metion what we should note about them.

This is the source code for `AbstractBaseUser`:

```python
class AbstractBaseUser(models.Model):
    password = models.CharField(_("password"), max_length=128)
    last_login = models.DateTimeField(_("last login"), blank=True, null=True)

    is_active = True

    REQUIRED_FIELDS = []

    # Stores the raw password if set_password() is called so that it can
    # be passed to password_changed() after the model is saved.
    _password = None

    class Meta:
        abstract = True

    def __str__(self):
        return self.get_username()

    def save(self, **kwargs):
        super().save(**kwargs)
        if self._password is not None:
            password_validation.password_changed(self._password, self)
            self._password = None

    def get_username(self):
        """Return the username for this User."""
        return getattr(self, self.USERNAME_FIELD)

    def clean(self):
        setattr(self, self.USERNAME_FIELD, self.normalize_username(self.get_username()))

    def natural_key(self):
        return (self.get_username(),)

    @property
    def is_anonymous(self):
        """
        Always return False. This is a way of comparing User objects to
        anonymous users.
        """
        return False

    @property
    def is_authenticated(self):
        """
        Always return True. This is a way to tell if the user has been
        authenticated in templates.
        """
        return True

    def set_password(self, raw_password):
        self.password = make_password(raw_password)
        self._password = raw_password

    def check_password(self, raw_password):
        """
        Returns a boolean of whether the raw_password was correct. Handles
        hashing formats behind the scenes.
        """

        def setter(raw_password):
            self.set_password(raw_password)
            # Password hash upgrades shouldn't be considered password changes.
            self._password = None
            self.save(update_fields=["password"])

        return check_password(raw_password, self.password, setter)

    async def acheck_password(self, raw_password):
        """See check_password()."""

        async def setter(raw_password):
            self.set_password(raw_password)
            # Password hash upgrades shouldn't be considered password changes.
            self._password = None
            await self.asave(update_fields=["password"])

        return await acheck_password(raw_password, self.password, setter)

    def set_unusable_password(self):
        # Set a value that will never be a valid hash
        self.password = make_password(None)

    def has_usable_password(self):
        """
        Return False if set_unusable_password() has been called for this user.
        """
        return is_password_usable(self.password)

    def get_session_auth_hash(self):
        """
        Return an HMAC of the password field.
        """
        return self._get_session_auth_hash()

    def get_session_auth_fallback_hash(self):
        for fallback_secret in settings.SECRET_KEY_FALLBACKS:
            yield self._get_session_auth_hash(secret=fallback_secret)

    def _get_session_auth_hash(self, secret=None):
        key_salt = "django.contrib.auth.models.AbstractBaseUser.get_session_auth_hash"
        return salted_hmac(
            key_salt,
            self.password,
            secret=secret,
            algorithm="sha256",
        ).hexdigest()

    @classmethod
    def get_email_field_name(cls):
        try:
            return cls.EMAIL_FIELD
        except AttributeError:
            return "email"

    @classmethod
    def normalize_username(cls, username):
        return (
            unicodedata.normalize("NFKC", username)
            if isinstance(username, str)
            else username
        )
```

Here are the most important parts of the above source code:

1. `password` and `last_login` fields are defined here so in our custom User model, we don't need to define them, we just inherit them:

```python
password = models.CharField(_("password"), max_length=128)
last_login = models.DateTimeField(_("last login"), blank=True, null=True)
```

2. As you see the REQUIRED_FIELDS attribute is set to an empty list by default:

```python
REQUIRED_FIELDS = []
```

So we need to override this attribute in our custom User model and provide the required fields that are prompted for when creating a user via 'createsuperuser'. Note that we don't need to include the unique identifier for authentication field (which is set by `USERNAME_FIELD` attribute, the `email` field in our case) and the password field in `REQUIRED_FIELDS` attribute. Django will ask for them automatically when creating a user via 'createsuperuser'.  
3. We need to set the `USERNAME_FIELD` attribute to `"email"` because the `get_username` method always uses this attribute to return what field is used as the unique identifier for authentication.

```python
def get_username(self):
    """Return the username for this User."""
    return getattr(self, self.USERNAME_FIELD)
```

4. The useful `set_password` method is provided here which receives the raw password, hashes it and adds the hashed password in the password field of User object (it also adds the raw password to `_password` attribute):

```python
def set_password(self, raw_password):
    self.password = make_password(raw_password)
    self._password = raw_password
```

We also have another useful method named `check_password` which recieves a raw password and compares it with hashed password stored in password field:

```python
def check_password(self, raw_password):
    """
    Returns a boolean of whether the raw_password was correct. Handles
    hashing formats behind the scenes.
    """

    def setter(raw_password):
        self.set_password(raw_password)
        # Password hash upgrades shouldn't be considered password changes.
        self._password = None
        self.save(update_fields=["password"])

    return check_password(raw_password, self.password, setter)
```

## Question:

In `BaseUserManager` we have this normal attribute:

```python
is_active = True
```

And in my custom User model we have this:

```python
is_active = models.BooleanField(default=True)
```

What is the difference between them? And in general what is the difference between a normal attribute and an attribute defined using models.SOMEField?

## Answer:

The two declarations for `is_active` serve **completely different purposes**, reflecting the difference between a Python class attribute and a Django database field.

### 1. The `BaseUserManager` Attribute (`is_active = True`)

The `is_active = True` attribute you see in the `AbstractBaseUser` source code is a **static Python class attribute** used for **internal logic and consistency**.

- **Location:** Inside `AbstractBaseUser` (which your `User` model inherits).
- **Purpose:** It acts as a **default class-level property** for methods like `is_authenticated` and `is_anonymous`, which are also defined in `AbstractBaseUser`.
  - **Crucially, this is NOT a database field.**
  - Since `is_active` must be present on all concrete user models (it's required by Django's authentication system), the base class provides a default value to ensure these property methods work correctly right out of the box, even before any specific fields are defined.

### 2. The Custom User Model Field (`is_active = models.BooleanField(...)`)

The definition in your custom `User` model creates the actual **database column** and is the value Django's authentication system ultimately reads.

- **Location:** Inside your custom `User` model.
- **Purpose:** It defines a **database column**.
  - It tells Django's ORM: "Create a boolean column named `is_active` in the database table."
  - The `default=True` argument ensures that every time a new user object is saved to the database (and you don't provide a value), the database sets the initial value for that column to `True`.

### Summary of the Difference

| Attribute       | `is_active = True` (in `AbstractBaseUser`)                                     | `is_active = models.BooleanField(default=True)` (in `User` model) |
| :-------------- | :----------------------------------------------------------------------------- | :---------------------------------------------------------------- |
| **Type**        | Static Python Class Attribute                                                  | Django Model Field (Database Column)                              |
| **Persistence** | **Non-Persistent:** Exists only in Python memory.                              | **Persistent:** Creates a column in the database.                 |
| **Usage**       | Used by base class methods (like `is_authenticated`) for internal consistency. | Stores the actual, mutable state of the user account.             |

---

### The General Difference: Normal Attributes vs. `models.Field`

The distinction between a normal attribute and a `models.Field` is one of the most fundamental concepts in Django.

#### A. Normal Attributes (Pure Python)

A normal class attribute (e.g., `MY_CONSTANT = 10`) or an instance attribute set in `__init__` or a method is **pure Python code**.

- **They are NOT** automatically saved to the database.
- **They are NOT** visible in Django Admin unless explicitly added.
- **They are NOT** handled by the ORM (Django's database layer).
- **Use Cases:** Storing constants, internal flags, or temporary calculation results on the Python object.

#### B. `models.Field` (ORM-Managed)

When you define an attribute as `models.CharField`, `models.BooleanField`, etc., you are using Django's **Object-Relational Mapper (ORM)**.

- **They ARE** automatically saved to and loaded from the database via SQL.
- **They ARE** automatically included in `makemigrations`.
- **They ARE** the entire structure Django uses to generate your database schema.
- **Use Cases:** Defining persistent, structured data for your application (e.g., `email`, `created_at`, `is_staff`).

In the case of `is_active`, the one in the base class is a placeholder for consistency, but the one you define is the real, functional database column. By defining the `models.BooleanField` in your `User` class, you **override** the non-persistent class attribute from `AbstractBaseUser` with a persistent database field.

## Question:

In this line from `AbstractBaseUser` source code:

```python
password = models.CharField(_("password"), max_length=128)
```

Why it has provided a string for the field as `_("password")`? I never used a string in my models fields.

## Answer:

The line `password = models.CharField(_("password"), max_length=128)` uses two related concepts that are crucial for high-quality, reusable Django code: the **verbose name** and **internationalization (i18n)**.

Here is the breakdown of the string `_("password")`:

### 1. The Purpose of the String: `verbose_name`

The first positional argument passed to any Django field is its **`verbose_name`**.

- **Your Code:** You likely omitted the `verbose_name` (e.g., `email = models.EmailField(unique=True)`). When you omit it, Django automatically derives the verbose name from the field's attribute name. For your `email` field, the verbose name defaults to **"email"**.
- **Django Source Code:** In the `AbstractBaseUser` source, the string `"password"` explicitly sets the field's verbose name to "password". This name is used by:
  - The **Django Admin site** (as the column header).
  - **Form/Serializer field labels** (if not explicitly customized).
  - **Error messages** (e.g., "The field **password** is required.").

### 2. The Meaning of `_()`: Internationalization

The unusual part is the function wrapping the string: **`_("password")`**.

The underscore `_()` is an alias for Django's translation function (`gettext` or `gettext_lazy`).

- **What it does:** Wrapping a string in `_()` marks that string as translatable. When a user accesses the site in a different language (e.g., Spanish), Django's translation system looks up the string `"password"` in its Spanish translation file (`.po` file) and renders the text as "contraseña."
- **Why Django Uses It:** Since the built-in `User` model is part of the Django framework and must support all languages, every user-facing string (like field names and error messages) must be marked for translation.

In summary, the line means:

> "Create a `CharField` for the database, set its maximum length to 128, and use the translatable string **'password'** as its human-readable name (verbose name)."

You don't typically need to use `_("...")` in your own models unless you plan to translate your application into multiple languages. If you were building a production application for a global audience, you should adopt this practice for all your model fields.

## Question:

The `last_login` field is defined in `AbstractBaseUser`:

```python
last_login = models.DateTimeField(_("last login"), blank=True, null=True)
```

But I don't see the logic to automatically update this field whenever a user logs in. I think the logic must exist in `Model` class that the `AbstractBaseUser` inherits from?

## Answer:

You are correct that the logic to update `last_login` is not defined in the `AbstractBaseUser` model itself, nor is it in the base `Model` class.

Instead, the logic is implemented via a **Django signal** that listens for a successful login event.

### The Mechanism: The `user_logged_in` Signal

Django's authentication system uses a **signal/receiver pattern** to automatically update the `last_login` field:

#### 1\. The Signal Emitter

When a user successfully authenticates (usually by calling `django.contrib.auth.authenticate` and then `login` in a view), Django's authentication app internally executes the following:

```python
# Internal Django code executed upon successful login
from django.contrib.auth.signals import user_logged_in
user_logged_in.send(sender=user.__class__, request=request, user=user)
```

This line **emits** the `user_logged_in` signal.

#### 2\. The Signal Receiver

The logic to update the timestamp is located in a **receiver function** that is automatically connected to the `user_logged_in` signal. This receiver is part of the `django.contrib.auth` app and is called **`update_last_login`**.

The receiver function performs two key steps:

1.  It checks if the user's `last_login` field needs updating (i.e., if the user object has the field).
2.  If so, it updates the field to the current time (`timezone.now()`) and saves the user object to the database using an `update_fields` optimization (to only save the `last_login` field).

### Summary

The process ensures that the `last_login` field is updated **outside of the model's logic** (`save()`, etc.) and instead happens as a **side effect** of the authentication process.

This design keeps the core `User` model clean and decouples the field definition from the update logic, making the authentication system modular and extensible.

##

---

---

---

---

And this is the source code for `PermissionsMixin`:

```python
class PermissionsMixin(models.Model):
    """
    Add the fields and methods necessary to support the Group and Permission
    models using the ModelBackend.
    """

    is_superuser = models.BooleanField(
        _("superuser status"),
        default=False,
        help_text=_(
            "Designates that this user has all permissions without "
            "explicitly assigning them."
        ),
    )
    groups = models.ManyToManyField(
        Group,
        verbose_name=_("groups"),
        blank=True,
        help_text=_(
            "The groups this user belongs to. A user will get all permissions "
            "granted to each of their groups."
        ),
        related_name="user_set",
        related_query_name="user",
    )
    user_permissions = models.ManyToManyField(
        Permission,
        verbose_name=_("user permissions"),
        blank=True,
        help_text=_("Specific permissions for this user."),
        related_name="user_set",
        related_query_name="user",
    )

    class Meta:
        abstract = True

    def get_user_permissions(self, obj=None):
        """
        Return a list of permission strings that this user has directly.
        Query all available auth backends. If an object is passed in,
        return only permissions matching this object.
        """
        return _user_get_permissions(self, obj, "user")

    async def aget_user_permissions(self, obj=None):
        """See get_user_permissions()"""
        return await _auser_get_permissions(self, obj, "user")

    def get_group_permissions(self, obj=None):
        """
        Return a list of permission strings that this user has through their
        groups. Query all available auth backends. If an object is passed in,
        return only permissions matching this object.
        """
        return _user_get_permissions(self, obj, "group")

    async def aget_group_permissions(self, obj=None):
        """See get_group_permissions()"""
        return await _auser_get_permissions(self, obj, "group")

    def get_all_permissions(self, obj=None):
        return _user_get_permissions(self, obj, "all")

    async def aget_all_permissions(self, obj=None):
        return await _auser_get_permissions(self, obj, "all")

    def has_perm(self, perm, obj=None):
        """
        Return True if the user has the specified permission. Query all
        available auth backends, but return immediately if any backend returns
        True. Thus, a user who has permission from a single auth backend is
        assumed to have permission in general. If an object is provided, check
        permissions for that object.
        """
        # Active superusers have all permissions.
        if self.is_active and self.is_superuser:
            return True

        # Otherwise we need to check the backends.
        return _user_has_perm(self, perm, obj)

    async def ahas_perm(self, perm, obj=None):
        """See has_perm()"""
        # Active superusers have all permissions.
        if self.is_active and self.is_superuser:
            return True

        # Otherwise we need to check the backends.
        return await _auser_has_perm(self, perm, obj)

    def has_perms(self, perm_list, obj=None):
        """
        Return True if the user has each of the specified permissions. If
        object is passed, check if the user has all required perms for it.
        """
        if not isinstance(perm_list, Iterable) or isinstance(perm_list, str):
            raise ValueError("perm_list must be an iterable of permissions.")
        return all(self.has_perm(perm, obj) for perm in perm_list)

    async def ahas_perms(self, perm_list, obj=None):
        """See has_perms()"""
        if not isinstance(perm_list, Iterable) or isinstance(perm_list, str):
            raise ValueError("perm_list must be an iterable of permissions.")
        for perm in perm_list:
            if not await self.ahas_perm(perm, obj):
                return False
        return True

    def has_module_perms(self, app_label):
        """
        Return True if the user has any permissions in the given app label.
        Use similar logic as has_perm(), above.
        """
        # Active superusers have all permissions.
        if self.is_active and self.is_superuser:
            return True

        return _user_has_module_perms(self, app_label)

    async def ahas_module_perms(self, app_label):
        """See has_module_perms()"""
        # Active superusers have all permissions.
        if self.is_active and self.is_superuser:
            return True

        return await _auser_has_module_perms(self, app_label)
```

The only note I want you to pay attention to is that this mixin adds a `is_superuser` database field.

---

---

---

This is the source code for `BaseUserManager`:

```python
class BaseUserManager(models.Manager):
    @classmethod
    def normalize_email(cls, email):
        """
        Normalize the email address by lowercasing the domain part of it.
        """
        email = email or ""
        try:
            email_name, domain_part = email.strip().rsplit("@", 1)
        except ValueError:
            pass
        else:
            email = email_name + "@" + domain_part.lower()
        return email

    def get_by_natural_key(self, username):
        return self.get(**{self.model.USERNAME_FIELD: username})

    async def aget_by_natural_key(self, username):
        return await self.aget(**{self.model.USERNAME_FIELD: username})
```

Here the most important note I want you to pay attention to is that this `BaseUserManager` is quite simple and the most important method it provides for us is `normalize_email` which normalizes the email address by lowercasing the domain part of it.

---

---

---

---

### Question:

For our custom user model, we inherited from `AbstractBaseUser`. Why do we need to inherit from it?

### Answer:

You need to inherit from `AbstractBaseUser` for your custom user model because it provides the **core authentication functionality** required by Django's auth system, while allowing you maximum flexibility to define your own fields.

`AbstractBaseUser` is the base class for implementing a fully custom user model. It includes the absolute minimum required for a user to be functional within Django's authentication framework.

It specifically provides:

- **Password Hashing:** It includes the `password` attribute and handles the password hashing process through methods like `set_password()`, which is crucial for secure authentication.
- **Last Login Tracking:** It provides the `last_login` attribute.
- **Required Attributes:** It ensures you define the essential attributes and methods needed for authentication, such as the `USERNAME_FIELD` and methods like `get_full_name()`, which are necessary for interacting with Django's built-in systems (like the admin interface).

By inheriting from `AbstractBaseUser`, you get the secure authentication plumbing without the standard fields like `username`, `first_name`, and `last_name` that come with `AbstractUser`. This allowed you to completely replace the `username` with **email** as your primary login identifier.

#### Why Not `AbstractUser`?

If you had inherited from `AbstractUser` (which inherits from `AbstractBaseUser` and `PermissionsMixin`), you would have gotten all of Django's default user fields (username, first_name, etc.) automatically. Since your goal was to completely remove the `username` and use `email` exclusively, you chose `AbstractBaseUser` to start with a blank slate for the field definitions.

**`AbstractBaseUser` inherits from `models.Model`**, which is the ultimate base class for any Django database model.

---

The reason you can treat your custom `User` model as a standard Django model (add `EmailField`, `CharField`, `Meta` class, etc.) is because the inheritance chain guarantees it:

$$\text{User} \longrightarrow \text{AbstractBaseUser} \longrightarrow \text{models.Model}$$

#### 1. `models.Model` Functionality

Because your custom `User` model ultimately inherits from `models.Model`, it gains all the standard database features:

- **Database Schema:** The ability to define fields (like `email`, `first_name`, and your custom `created_at` field), which are then mapped to columns in your database table via migrations.
- **Model Management:** Access to the `objects` manager for data manipulation (creation, retrieval, update, deletion).
- **Django ORM:** Full integration with the Object-Relational Mapper (ORM), allowing you to interact with your user data using Python objects and querysets.

#### 2. `AbstractBaseUser` Functionality

On top of the standard model features, **`AbstractBaseUser` overlays the essential authentication features** by providing:

- **Security:** Handles password hashing and storage (`password` field and `set_password()` method).
- **Core Authentication Plumbing:** Provides the infrastructure for `last_login`, the `USERNAME_FIELD` setting, and the methods required by Django's authentication backend.
- **Flexibility:** Allows you to define all your non-essential fields from scratch (like removing the default `username` and adding `first_name`, `last_name`, etc.).

In short, **inheriting from `AbstractBaseUser` gives you the best of both worlds**: all the capabilities of a standard Django model _plus_ the necessary security and framework integration for authentication.

Note: As we mentioned above, by inheriting from `AbstractBaseUser`, we get a `last_login` field.
When a user successfully logs in using one of Django's built-in authentication views (or any process that calls Django's login function, which is what rest_framework_simplejwt often uses under the hood), the last_login field is updated automatically.

You don't have to do anything in your custom view or serializer to manage this field.

---

#### Core Features Provided by `AbstractBaseUser`

`AbstractBaseUser` provides the following **essential functionality and attributes**:

#### Essential Attributes and Methods (Security & State)

1.  **`password` attribute:** The field used to store the secure hash of the user's password.
2.  **`set_password()` method:** A crucial method that handles **hashing the raw password** and setting the `password` attribute safely.
3.  **`last_login` attribute:** A `DateTimeField` that records when the user last logged in.
4.  **Automatic `last_login` update:** Handled automatically by Django's signal system upon successful login, as you noted.
5.  **`is_authenticated` property:** A read-only property that always returns `True` for active user objects, required by Django's authentication system.
6.  **`is_anonymous` property:** A read-only property that always returns `False` for user objects (used to distinguish them from the `AnonymousUser` object).

---

#### Configuration and Plumbing

7.  **`USERNAME_FIELD` setting:** This is the **required class attribute** that tells Django which unique field (like your `email`) should be used as the identifier for logging in.
8.  **`get_username()` method:** A method required by the framework that returns the value of the field specified by $`USERNAME\_FIELD`$.
9.  **`REQUIRED_FIELDS` setting:** This is the **required class attribute** that lists fields that must be prompted for when creating a user via the `createsuperuser` command. (While this is defined on your model, its presence is mandated by the `AbstractBaseUser` interface).

These minimal components allow your custom model to plug directly into Django's powerful authentication backend while leaving all other fields (like `first_name`, `is_staff`, etc.) for you to define.

---

Plumbing has two primary meanings: a literal one related to pipes and water, and a metaphorical one used in software engineering and general systems.

#### 1. Literal Meaning (Pipes and Water) 💧

**Plumbing** literally refers to the system of **pipes, tanks, fittings, valves, and fixtures** installed for the distribution of water and the disposal of sewage or waste fluid in a building.

---

#### 2. Metaphorical Meaning (Software/Systems)

In a metaphorical or technical context, especially in software engineering (as used when discussing Django authentication):

**Plumbing** refers to the **hidden, foundational infrastructure, or low-level mechanisms** that allow a system to work, but which a typical user or higher-level developer rarely interacts with directly.

It represents the **necessary, behind-the-scenes connections, integrations, and boilerplate code** that links different components together.

#### Example in Programming

When discussing the Django User Model:

- The **"plumbing"** provided by `AbstractBaseUser` is the **automatic, low-level logic** (like password hashing, the `last_login` signal, and the core authentication checks) that makes the custom model compatible with the rest of the Django framework.
- This plumbing is essential, but you don't need to write or manage the code for it; you just inherit it.

---

### Question:

We talked about what inheriting from `AbstractBaseUser` provides to our custom user model. Next tell me what inheriting from `PermissionsMixin` provides to my custom user model. Also teach me a little about mixins in python.

### Answer:

The primary purpose of inheriting from `PermissionsMixin` is to provide your custom user model with the **necessary fields and methods for Django's authorization (permission) system**.

Without it, your users wouldn't be able to interact with Django's administrative and permission features.

It gives you the following fields and methods:

| Feature     | Provided By `PermissionsMixin`                          | Purpose                                                                                                                                    |
| :---------- | :------------------------------------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------- |
| **Field**   | `is_superuser`                                          | A boolean field that designates this user as having all permissions without explicitly being assigned them.                                |
| **Fields**  | `groups`                                                | A Many-to-Many field linking the user to Django's built-in `Group` model.                                                                  |
| **Fields**  | `user_permissions`                                      | A Many-to-Many field linking the user to specific, individual permissions.                                                                 |
| **Methods** | `has_perm()`, `has_perms()`, `has_module_perms()`, etc. | Methods used by the framework to check if the user has a specific permission, group membership, or permissions for a specific application. |

**In short:** If you want your users to be able to log into the **Django Admin**, have **staff permissions**, and/or be assigned specific **permissions** in your application, you must include `PermissionsMixin`.

---

#### What is a Mixin in Python?

A **Mixin** is a class designed to be inherited by other classes, but not intended to be instantiated on its own. Its purpose is to **"mix in"** methods and properties to enhance the functionality of another class, thereby providing a form of code reuse.

##### Key Characteristics of a Mixin:

1.  **Code Reusability:** It encapsulates a specific set of behavior (like permission checking) that can be easily added to multiple, unrelated classes.
2.  **No Standalone Use:** You would never create an object directly from `PermissionsMixin`. It only works as a component of another class (like your `User` model).
3.  **Horizontal Enhancement:** Unlike normal inheritance, which establishes an "is-a" relationship (e.g., a "Dog" is an "Animal"), a mixin establishes a "can-do" or "has-a" relationship (e.g., your `User` **has** permission-checking abilities).

In your custom model's definition:

```python
class User(AbstractBaseUser, PermissionsMixin):
    # ...
```

Your `User` class **inherits from two classes**:

1.  **`AbstractBaseUser`:** Provides the essential **authentication** plumbing (the "is-a" part of being a user).
2.  **`PermissionsMixin`:** Provides the essential **authorization** plumbing (the "can-do" part of checking permissions).

---

A few questions about the `REQUIRED_FIELDS` setting:

---

### Question 1:

What happens if I comment this `REQUIRED\_FIELDS = ["first\_name", "last\_name"]` out?

### Answer:

If you comment out or remove `REQUIRED\_FIELDS = ["first\_name", "last\_name"]`, the only change you'll notice is when you run the **`createsuperuser`** management command.

When you run `py manage.py createsuperuser`:

- **Django will no longer prompt you** to enter values for `first_name` and `last_name`.
- The `first_name` and `last_name` fields in the database will be saved as **empty strings** (`''`), since you defined them as $`blank=True`$ in your model.

**For normal user creation (e.g., via a registration form or API):** Nothing changes. This setting **only affects** the `createsuperuser` command and has no impact on other model operations like creating users through the `User.objects.create_user()` manager method or serializers.

---

### Question 2:

Why `email` and `password` aren't in `REQUIRED_FIELDS`?

### Answer:

You don't need to put `email` or `password` in the `REQUIRED_FIELDS` list.

This is because:

- **Authentication Identifier (`email`):** The field defined by `USERNAME_FIELD` (which is `'email'` in your case) is **always** prompted for first by the `createsuperuser` command. It is considered the primary, non-optional identifier.
- **Password:** The `password` is **always** prompted for last by the `createsuperuser` command for security reasons, regardless of any field lists.

The `REQUIRED_FIELDS` list is only for **other, non-primary fields** that you want the `createsuperuser` command to explicitly prompt for during the setup process.

---

### Question 3:

Is this setting unique to `AbstractBaseUser`?

### Answer:

Yes, this setting is specific to Django's authentication system and is required when inheriting from `AbstractBaseUser` (or `AbstractUser`).

- **Normal Models:** You **do not** have a `REQUIRED_FIELDS` setting in normal models inheriting only from `models.Model`. The concept doesn't apply because normal models aren't used by the `createsuperuser` command.
- **User Models:** `AbstractBaseUser` dictates the need for both `USERNAME_FIELD` and `REQUIRED_FIELDS` to define the minimal data structure necessary to create an administrative user through the command line.

---

### Question:

Okay let's go to the custom user model manager and try to understand it. In this code, in models.py file we see a custom model manager. I know that custom managers are a general Django concept, and they're not just used for creating a custom user model. We can define a manager for every model. So first tell me what is the most typical use case for creating custom model managers instead of the default `objects` manager?

### Answer:

The most common use-case for creating custom model managers is to **define reusable, table-level query methods (QuerySets)**.

A custom manager allows you to define methods that return a filtered or modified queryset, which can then be called across your application as a concise, readable expression. This prevents you from repeating the same filtering logic (like complex `filter()`, `exclude()`, or `annotate()` calls) in multiple views or utility functions.

#### Example

Imagine you have a `Post` model. You could define a custom manager to easily retrieve different subsets of posts:

```python
class PostManager(models.Manager):
    def published(self):
        """Returns all posts that are published and not soft-deleted."""
        return self.filter(status='PUBLISHED', deleted=False).order_by('-published_date')

    def featured(self):
        """Returns only the top 5 featured posts."""
        return self.published().filter(is_featured=True)[:5]

class Post(models.Model):
    # ... fields
    objects = PostManager()
```

Instead of repeating the complex `filter()` calls every time, you can now write simple, expressive code:

- To get all published posts:
  ```python
  published_posts = Post.objects.published()
  ```
- To get the 5 featured posts:
  ```python
  featured_list = Post.objects.featured()
  ```

This approach promotes the **Don't Repeat Yourself (DRY)** principle, making your code cleaner, easier to maintain, and more readable.

Note that our PostManager is inheriting from `models.Manager` so we'll have access to all the methods on the default manager like `all()`, `get()`, `filter()` etc, plus the methods we defined ourselves like `published()`.

---

### Question:

In the custom user manager, `UserManager`, why we are inheriting from `BaseUserManager` and what it provides to our manager? Normally when creating a manager for a model we inherit from `models.Manager`, so explain to me what this `BaseUserManager` provides more.

### Answer:

You are inheriting from **`BaseUserManager`** instead of the standard `models.Manager` because **`BaseUserManager` is specifically designed to handle the creation and management of users in Django's authentication framework**.

`BaseUserManager` inherits from `models.Manager`, so it provides all the standard ORM functionality (like `.filter()`, `.all()`, etc.), but it **adds essential methods and utilities** required for managing users securely.

| Feature                       | Provided by `BaseUserManager`                                                                                                                                          | Purpose in Your Custom Manager                                                                                                                             |
| :---------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **User Creation Shell**       | Provides the basic structure for `create_user()` and `create_superuser()`.                                                                                             | You **must** override these to enforce your custom rules (like using `email` as the identifier) and properly call the `set_password()` method.             |
| **Email Normalization**       | Provides the **`normalize_email(email)`** utility method.                                                                                                              | This standardizes the email address (e.g., lowercase) before saving, which is crucial for consistent and case-insensitive login.                           |
| **Password Hashing Plumbing** | It ensures that when you call `user.set_password(password)` inside your custom creation methods, the user object is correctly prepared for saving the hashed password. | This is part of the "plumbing" that links the manager's creation logic to the `AbstractBaseUser`'s password security features.                             |
| **Integration**               | It integrates seamlessly with Django's core authentication and administrative system.                                                                                  | When you run `py manage.py createsuperuser`, the system knows to look for and rely on the methods provided by a manager inheriting from `BaseUserManager`. |

#### Summary

If you only inherited from `models.Manager`, your custom manager would lack the specialized methods and utility functions needed to correctly handle the **security and authentication requirements** for creating users and superusers. `BaseUserManager` is the intermediary class that adds this necessary authentication intelligence.

---

### Question:

I'm new to Python. Teach me about the Python concepts of `**extra_fields`, `setdefault`, and `get` on a dictionary which are used in `create_superuser` method.

### Answer:

This `create_superuser` method is crucial for securely creating a `Superuser` account (or `Super-admin` account or `God mode` account). Let's break down the Python concepts of `**extra_fields`, `setdefault`, and `get` on a dictionary.

#### Python Basics: `**extra_fields`

The term `**extra_fields` in the method signature (`def create_superuser(self, email, password, **extra_fields):`) is Python syntax for handling **keyword arguments**.

1.  **What it Does:** It tells the method to accept any **unexpected keyword arguments** passed to it (arguments other than `self`, `email`, and `password`).
2.  **How it Works:** Python collects all those extra, named arguments into a single dictionary named `extra_fields`.
3.  **The Benefit:** It makes the method flexible. When you run `createsuperuser`, Django might pass fields like `first_name` and `last_name` that you required. `**extra_fields` catches all these values.

#### Example of `**extra_fields`

If someone called the method like this:

```python
# The call (not what manage.py does, but illustrative)
manager.create_superuser('admin@blog.com', 'mypass', first_name='Admin', role='owner')
```

The resulting `extra_fields` dictionary inside the method would be:

```python
{'first_name': 'Admin', 'role': 'owner'}
```

---

#### Dictionary Methods: `setdefault` and `get`

In your `create_superuser` method, `extra_fields` is treated as a standard Python dictionary, and it uses two powerful dictionary methods:

##### 1\. `setdefault()`

This is used to ensure certain fields are present with a specific value, but **only if they haven't already been provided**.

```python
extra_fields.setdefault("is_staff", True)
```

- **Logic:** Look in the `extra_fields` dictionary for the key `"is_staff"`.
  - If `"is_staff"` **is not there**, add it and set its value to `True`.
  - If `"is_staff"` **is already there**, do nothing and keep its existing value.
- **Purpose Here:** When creating a superuser, these three permissions **must** be `True`. `setdefault` ensures that even if the command doesn't explicitly provide them, they are set correctly before the user is saved.

#### 2\. `get()`

This is a safe way to check the value of a key in a dictionary without risking an error if the key doesn't exist. When accessing a value in a dictionary with bracket syntax (dict[key]), if the key doesn't exist, an error will be raised.

The main advantage of using dict.get(key, default) over the standard bracket notation (dict[key]) is its ability to specify a default value to return if the key is not found in the dictionary. The get() method handles missing keys gracefully by returning a default value instead of raising an error.

If you call get(key) with only one argument and the key is missing, it returns None.

```python
if extra_fields.get("is_staff") is not True:
    raise ValueError("Superuser must have is_staff=True.")
```

- **Logic:** Look for the key `"is_staff"` in the dictionary.
  - If `"is_staff"` **is found**, return its value.
  - If `"is_staff"` **is not found**, return a default value (by default, `None`).
- **Purpose Here:** This line is a **safety check**. Even though we used `setdefault` to set these fields to `True`, this check is a final guard to prevent any external code or override from accidentally setting the crucial superuser permissions to `False`.

---

### Question:

In the `UserManager`, in `create_user` method we have this line: `user.set_password(password)`. I think the `set_password` method accepts the raw password, hashes the raw password and finally adds this value to the password field of the model instance. Am I correct?

### Answer:

Yes, that is exactly correct.

The line `user.set_password(password)` performs the crucial security steps for handling the user's password:

1.  **Hashing:** It takes the **raw, plain-text password** (the string passed to the method).
2.  **Security:** It processes this raw password through Django's configured, industry-standard **hashing algorithm** (e.g., PBKDF2) along with a **salt** (random data) to create a secure, one-way hash.
3.  **Storage:** It then sets the user object's `password` attribute to this newly generated **hashed value**.
4.  **Database Save:** The password hash is saved to the database when you call $`user.save()`$ on the next line.

This ensures that the raw password is **never stored** in the database, which is fundamental to secure authentication.

---

### Question:

In every model manager we have a `create` method we use like: `Product.objects.create()`. Here in the user serializer, we have defined a create method too. My question is, are these two create methods the same? Or they have different purposes?

### Answer:

No, the two `create` methods you mentioned are **not the same** and serve fundamentally different purposes within the Django and Django REST Framework (DRF) structure.

They operate at different layers of your application.

#### 1\. Model Manager `create()` (`Product.objects.create()`)

The `create()` method on a model manager (like `models.Manager` or your custom `UserManager`) is a **database-level operation**.

- **Purpose:** It is a shorthand method that combines creating a model instance (`__init__`) and immediately saving it to the database (`.save()`) in a single step.
- **Layer:** Django ORM (Database Layer).
- **Authentication Context:** When dealing with a custom user model, you typically avoid the default `User.objects.create()` and use your custom **`User.objects.create_user()`** instead. This is because the default `create()` method would not correctly hash the password, while your custom `create_user()` method ensures that `set_password()` is called before saving.

---

#### 2\. Serializer `create()` (`UserSerializer.create()`)

The `create()` method on a DRF serializer is an **API/Data Validation layer operation**.

- **Purpose:** Its responsibility is to define **how a validated, incoming Python dictionary (API request data) should be turned into a saved model instance.** This method only runs _after_ the data has passed all validation checks defined by the serializer (e.g., ensuring the email format is valid, max lengths are respected, etc.).

- **Layer:** Django REST Framework (Serialization Layer).

- **Your Code's Role:** Your serializer's `create()` method acts as the **bridge**. It correctly extracts the sensitive `password` and other data from the API payload and delegates the actual secure object creation to the model manager's specialized method:

  ```python
  # Serializer's job: Process validated API data
  def create(self, validated_data):
      email = validated_data.pop("email")
      password = validated_data.pop("password")

      # Delegating the database/security work to the UserManager
      user = User.objects.create_user(email, password, **validated_data)
      return user
  ```

---

#### Summary of Differences

| Feature         | Model Manager `create()` (or `create_user()`)                       | Serializer `create()`                                                    |
| :-------------- | :------------------------------------------------------------------ | :----------------------------------------------------------------------- |
| **Location**    | Defined in `models.Manager` or `BaseUserManager`                    | Defined in `serializers.Serializer` or `ModelSerializer`                 |
| **Input**       | Raw Python arguments/keywords                                       | `validated_data` (A clean, validated Python dictionary)                  |
| **Function**    | **Saves data to the database** (and handles security like hashing). | **Converts API data into a model instance** by calling a manager method. |
| **When Called** | Directly in any Django code, view, or shell.                        | Automatically by the DRF ViewSet/APIView when processing a POST request. |

---

### Question:

Every serializer in DRF has default `create` and `update` methods. These methods are there because we inherit from `serializers.ModelSerializer`. In create method, the serializer uses the create method of the model manager. We'll have a default create method on our custom model manager too because when creating the custom model manager, we inherited from `BaseUserManager` which in turn inherits from `models.Manager` itself. So we'll have a default create method. Here we don't want the serializer to use the create method in our custom user manager (which a serializer normally does) because the default create method on our custom model manager doesn't hash the password. So we needed to override the create method in our serializer and tell it to use `create_user` method instead of `create` method in our custom user model manager. Right?

### Answer:

You are **correct**.

Your conclusion is **100% correct**.

Here is the refined explanation:

#### Breakdown of the Statement

| Your Statement                                                                                                                                                                                                    | Status                  | Clarification                                                                                                                                                                                                 |
| :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **"Every serializer in DRF has default create and update methods. These methods are there because we inherit from `serializers.ModelSerializer`."**                                                               | **Correct**             | `ModelSerializer` provides these methods to handle standard model CRUD operations automatically.                                                                                                              |
| **"We'll have a default create method on our custom model manager too because when creating the custom model manager, we inherited from `BaseUserManager` which in turn inherits from `models.Manager` itself."** | **Technically Correct** | Yes, the `create()` method is inherited from `models.Manager`, so it technically exists on `User.objects`.                                                                                                    |
| **"In create method, the serializer uses the default create method of the model manager."**                                                                                                                       | **Correct**             | The default `ModelSerializer.create()` logic _will_ attempt to call `User.objects.create(**validated_data)`.                                                                                                  |
| **"Here we don't want the serializer to use the create method on our custom user manager because it doesn't hash the password."**                                                                                 | **Crucially Correct**   | This is the **primary security reason** for the override. The default `User.objects.create()` inherited from `models.Manager` does not know about the password hashing logic and would save the raw password. |
| **"So we needed to override the create method in serializer and tell it to use `create_user` method instead of `create` method in our custom user model manager."**                                               | **Correct**             | By overriding the serializer's `create()`, you force it to use the secure, custom **`User.objects.create_user()`** method that you specifically wrote to include `user.set_password()`.                       |

#### Summary of the "Why"

You override the serializer's `create()` method **specifically to bypass the insecure default behavior** of the `ModelSerializer`. The default serializer calls the **default `create()` method** on the manager, which is inherited from the general `models.Manager` and is **not secure** for user models.

Your override ensures the secure method—**`create_user()`**—is the only one used for registration, guaranteeing that the password is properly hashed.

---

### Question:

In `update` method of the user serializer, we have this line:
`instance = super().update(instance, validated_data)`

what is this `super()` and why it has an update method?

### Answer:

That line of code uses Python's `super()` function to call the default `update()` method of the **parent class** in the inheritance hierarchy. This is crucial for securely handling updates in your serializer.

---

#### 1\. What is `super()`?

In Python, `super()` is a built-in function that provides a way to access methods of the parent class (or sibling classes, in more complex inheritance) from within a class that inherits from it.

In the context of your `UserSerializer`:

- Your `UserSerializer` inherits from **`rest_framework.serializers.ModelSerializer`**.
- When you call `super().update(...)`, you are telling Python to look for and execute the `update` method defined in the **`ModelSerializer`** class.

---

#### 2\. Why Does `super()` Have an `update()` Method?

The parent class, **`ModelSerializer`**, has a default `update()` method that handles all the standard, non-password field updates.

The **default `ModelSerializer.update()` method** is responsible for:

1.  **Iterating:** It loops through every key-value pair in the `validated_data` dictionary.
2.  **Assignment:** For each item, it sets the corresponding attribute on the `instance` (your `User` object).
3.  **Saving:** It calls `instance.save()` to persist the changes to the database.

#### Why You Use `super().update()` in Your Custom Method

You needed to override the `update()` method in your `UserSerializer` for one specific reason: **to securely handle the password change**.

Your custom logic is:

1.  **Extract the password:** You safely remove the `password` from `validated_data`.
    ```python
    password = validated_data.pop("password", None)
    ```
2.  **Call the Parent:** You use `super().update(instance, validated_data)` to let the parent class handle **all the other fields** (`first_name`, `last_name`, `is_staff`, etc.) automatically.
    ```python
    instance = super().update(instance, validated_data)
    ```
3.  **Securely Set Password:** You only handle the `password` separately, ensuring it gets hashed via `instance.set_password(password)` if it was provided.

This approach follows the principle of **"Don't Repeat Yourself" (DRY)**: you only write the custom logic (password hashing) that you need, and you delegate the standard, repetitive logic (updating all other fields) back to the reliable base class.

---

### Question:

You said we use `super()` to access the parent class. Our serializer is the child class, so it should inherit all the methods from the parent class, including the `update` method. So instead of `instance = super().update(instance, validated_data)`, why we can't use `instance = self.update(instance, validated_data)`?

I think the reason is we have over-ride the `update` method and we can't use it inside itself. That creates a loop.

### Answer:

That is **exactly right**! You've hit upon the core reason and a fundamental concept of object-oriented programming (OOP): **Recursion and Method Overriding**.

---

#### The Problem with `self.update()`

You correctly identified that if you used `self.update(instance, validated_data)` inside the custom `update` method of your `UserSerializer`, you would create an **infinite recursive loop**, leading to a stack overflow error.

Here's why:

1.  A request comes in to update a user.
2.  The API framework calls your custom `UserSerializer.update()`.
3.  Inside your custom method, the code sees `self.update(...)`.
4.  `self.update(...)` is a call to the **very same method** (step 2).
5.  Steps 2, 3, and 4 repeat endlessly until the program runs out of memory.

#### The Solution: `super().update()`

The purpose of **method overriding** is to create specialized behavior while still reusing the general behavior of the parent.

The `super().update(instance, validated_data)` call serves two critical functions:

1.  **Bypassing the Child:** It explicitly tells Python: "Do **not** call the overridden `update` method on this instance (`self`). Instead, go up one level to the parent class (`ModelSerializer`) and execute _its_ version of the `update` method."
2.  **Delegating Standard Work:** It efficiently delegates the standard, secure field updating (for `first_name`, `is_staff`, etc.) back to the base `ModelSerializer` while you only handle the special case (the password) in your custom code.

This pattern is a hallmark of good object-oriented design, allowing you to **extend** a class's functionality without breaking or rewriting its core logic.

---

### Question:

Why we needed to include the `is_staff` field in our custom user model, but didn't need to include `is_superuser` field?

### Answer:

In your custom user model definition, you explicitly included the `is_staff` field, but you **did not need to explicitly include the `is_superuser` field** because your model inherits from **`PermissionsMixin`**.

#### The Role of `PermissionsMixin`

When you define your custom user model:

```python
class CustomUser(AbstractBaseUser, PermissionsMixin):
    # ... your fields
    is_staff = models.BooleanField(default=False)
    # ...
```

1.  **`PermissionsMixin` provides `is_superuser`:** The `PermissionsMixin` is a class that automatically includes all the fields necessary for Django's comprehensive permission system, including the `is_superuser` boolean field, the methods to check permissions, and the required fields for groups/user permissions (like `groups` and `user_permissions`).

2.  **`is_staff` is a Prerequisite:** While `PermissionsMixin` provides the permission logic, **Django requires you to define the `is_staff` field yourself** because it's the fundamental access control mechanism for the Admin site, and it's not automatically included by `AbstractBaseUser`. You must declare it as a simple boolean field.

3.  **The Manager Handles the Setting:** Even though you didn't declare `is_superuser` in your model, the field exists in the database table, and your `CustomUserManager` handles setting it:

    ```python
    extra_fields.setdefault("is_superuser", True)
    ```

In short, you only had to declare `is_staff` and other custom fields, while relying on the powerful **`PermissionsMixin`** to automatically inject the required **`is_superuser`** and related permission fields into your database table.

---

### Note:

The **`is_active` field is necessary and should not be removed**. It's a fundamental part of the Django authentication system.

Here is the brief explanation of why:

#### Why `is_active` is Necessary

The `is_active` field is a core boolean flag used by Django's authentication backend to determine if a user **is allowed to log in**.

1.  **Authentication Check:** During the login process, Django's backend checks if a user exists and if their password matches. The **final check is always the `is_active` field**. If `is_active` is set to `False`, the login attempt will fail, even if the password is correct.
2.  **Disabling Accounts:** It's the standard, non-destructive way to **deactivate or ban a user's account** without permanently deleting their data from the database. A site administrator can simply uncheck this box in the Django Admin to instantly revoke login privileges.
3.  **Required by `AbstractBaseUser` Logic:** Although `AbstractBaseUser` doesn't strictly _define_ the field, the underlying authentication logic that it relies on **expects this field to exist** on the model to function correctly. While you could technically implement your own authentication backend that bypasses this check, it's strongly advised to stick to the standard model structure.

**Conclusion:** Keep the `is_active` field.

---

### Note:

The line `REQUIRED_FIELDS = ["first_name", "last_name"]` is a **model-level setting** that dictates which fields, besides the primary identifier and password, must be provided when creating a superuser via the management command.

Here is what it does:

- **Applies to:** The `python manage.py createsuperuser` command **only**.
- **Purpose:** It instructs Django to **prompt the user** for values for `"first_name"` and `"last_name"` during the interactive superuser creation process.
- **The Default Requirements:** When creating any user via the terminal, Django automatically asks for:
  1.  The field defined by **`USERNAME_FIELD`** (which is `'email'` in your model).
  2.  The user's **password**.
- **The Role of `REQUIRED_FIELDS`:** This list allows you to **add additional, non-default fields** that must be supplied to the `create_superuser` method.

**In brief:** It makes `first_name` and `last_name` **mandatory only** when you run the `createsuperuser` command, ensuring that all necessary data for an administrator account is collected upfront.

---

### Note:

Defining the `get_full_name()` and `get_short_name()` methods on your custom user model is **optional** when inheriting from `AbstractBaseUser`.

While not strictly required for the database or basic authentication, these methods are often included for two main reasons:

1.  **Django Convention:** The built-in (default) `User` model includes these methods. By adding them to your custom model, you maintain **compatibility** and **consistency** with the standard Django API.
2.  **External Package Compatibility:** Many third-party packages (especially those related to the Django Admin, notifications, or user profile display) rely on the existence of these methods to retrieve a user's display name. Including them ensures your custom model works seamlessly with these tools.

In short, you can remove them if you are _absolutely certain_ you won't need to display a user's full or short name anywhere using the standard Django methods, but keeping them is **best practice**.

---

### Note:

About `Meta` class, these lines define the **metadata** for your `CustomUser` model. This metadata does **not** affect how the model works in the database or during authentication; it only controls how the model is displayed to humans, primarily in the **Django Admin** interface.

Here is a brief breakdown of what each line does:

---

#### Model Metadata (`class Meta`)

#### 1\. `verbose_name`

```python
verbose_name = "user"
```

- **What it does:** Sets the **singular, human-readable name** for the model.
- **Where it appears:** It appears when referring to a single instance of the model, such as the title of the page when editing a single user in the Django Admin (e.g., "Change **user**").

#### 2\. `verbose_name_plural`

```python
verbose_name_plural = "users"
```

- **What it does:** Sets the **plural, human-readable name** for the model.
- **Where it appears:** It appears in lists and headings, such as the entry in the Django Admin sidebar list (e.g., "**Users**") and the title of the page listing all users ("Select **users** to change").

If you omit these lines, Django will automatically generate the verbose names from the model's class name (`CustomUser`), which might look less clean (e.g., it might display as "Custom user" or "Custom users"). Setting them explicitly ensures the text is clear and professional.

---

### Question:

I think in serializers, the serializer arguments determine the behavior of the serializer.

### Answer:

You are fundamentally **correct** in your understanding of how a DRF serializer's behavior is determined by the arguments passed to its constructor.

The serializer object acts as a central **data transformer and validator**, and its mode of operation (read, create, or update) is indeed set by the arguments provided at instantiation.

---

#### How Serializer Arguments Determine Behavior

| Request Type               | Serializer Instantiation                                | Primary Argument(s)                               | Behavior                                                                                                                    |
| :------------------------- | :------------------------------------------------------ | :------------------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------- |
| **GET (Read)**             | `Serializer(instance)`                                  | Only a **model instance** (or QuerySet).          | Serializes the instance data into JSON/API output.                                                                          |
| **POST (Create)**          | `Serializer(data=request.data)`                         | Only **`data`** (the incoming request payload).   | Runs validation; on `.save()`, calls the internal **`create()`** method.                                                    |
| **PUT (Full Update)**      | `Serializer(instance, data=request.data)`               | **Instance** and **`data`**.                      | Runs validation; on `.save()`, calls the internal **`update()`** method. Requires all required fields in the data.          |
| **PATCH (Partial Update)** | `Serializer(instance, data=request.data, partial=True)` | **Instance**, **`data`**, and **`partial=True`**. | Runs validation; on `.save()`, calls the internal **`update()`** method. Only requires fields that are present in the data. |

#### Key Takeaways

1.  **Read Mode vs. Write Mode:**

    - If you pass a model instance (or QuerySet) as the **first positional argument**, the serializer is primarily in **read mode** (serialization).
    - If you pass the `data=` keyword argument, the serializer switches to **write mode** (deserialization and validation).

2.  **Create Mode vs. Update Mode:**

    - When in write mode, the absence of a model instance (the first positional argument) signals a **creation** operation (calls `create()` on `.save()`).
    - When in write mode, the presence of a model instance (the first positional argument) signals an **update** operation (calls `update()` on `.save()`).

3.  **Patch Mode:** The `partial=True` keyword argument is a flag that modifies the **validation** behavior during an update, allowing required fields to be omitted if they aren't part of the request payload. It still uses the **`update()`** method for saving.

---

### Question:

What does `serializer.data` contain in every case of your answer above?

### Answer:

The meaning and content of `serializer.data` depends entirely on whether the serializer instance is being used for **reading (serialization)** or **writing (validation/deserialization)**.

---

#### 1\. When Reading Data (GET Requests)

In a GET request, when you initialize the serializer with a model instance (or a QuerySet), `serializer.data` represents the **output data** that is ready to be sent back to the client, typically as JSON.

- **Role:** Serialization (Python objects $\longrightarrow$ Native Python data types, ready for JSON rendering).
- **Content:** A Python dictionary (or a list of dictionaries) where the keys match the fields defined in your serializer's `Meta.fields`. The values are the corresponding values pulled from the database instance.
- **Example:**
  ```python
  serializer = UserSerializer(user)
  # serializer.data will be:
  # {'id': 1, 'email': 'user@example.com', 'full_name': 'John Doe', ...}
  ```

---

#### 2\. When Writing Data (POST, PUT, PATCH Requests)

In write requests (POST, PUT, PATCH), when you initialize the serializer with the `data=` keyword argument, `serializer.data` holds the **validated data** after the `.is_valid()` method has been called.

- **Role:** Deserialization (Validation $\longrightarrow$ Object Creation/Update $\longrightarrow$ Serialization of Result).
- **Content:**
  - **Before `.save()`:** If you check `serializer.data` immediately after `serializer.is_valid()`, it contains the **validated data** used to create or update the object.
  - **After `.save()`:** It contains the **final, serialized representation of the newly created or updated model instance**. This output is generated by implicitly running the serializer in read mode on the saved instance.
- **Example (POST Request):**
  ```python
  serializer = UserSerializer(data={'email': 'new@user.com', ...})
  if serializer.is_valid():
      serializer.save() # User object is created and saved
      # serializer.data will now be the serialized representation of the NEW user:
      # {'id': 5, 'email': 'new@user.com', 'full_name': 'New User', ...}
  ```

#### Summary

| State                                             | Purpose          | Content of `serializer.data`                                                              |
| :------------------------------------------------ | :--------------- | :---------------------------------------------------------------------------------------- |
| **Read (GET)**                                    | Output to Client | **Serialized model data**                                                                 |
| **Write (POST/PUT/PATCH) After `.save()`**        | Output to Client | **Serialized newly created/updated model instance**                                       |
| **Write (After `.is_valid()`, Before `.save()`)** | Internal Use     | **Validated, clean input data** (`serializer.validated_data` is generally preferred here) |

---

### Question:

I sent a `PUT` request using `Insomnia` to `/api/users/me`. I only provided a new email in the request body and the email field got updated successfully and other fields remained untouched. You said before that in a PUT request, we update the whole object in the database. I just provided the email and only the email field got updated. I expected the other fields I didn't provide in my request body would be set to `None`.

### Answer:

That's an excellent observation that highlights a crucial difference between the standard **HTTP PUT** definition and how **Django Rest Framework (DRF) serializers** handle updates\!

Your expectation is correct based on the pure REST/HTTP specification, but DRF's `ModelSerializer` implementation for updating data modifies this behavior for convenience and practicality.

Here's why only the email was updated and the other fields weren't wiped out:

#### DRF's Behavior for `PUT` with `ModelSerializer`

When you use a **`ModelSerializer`** for updates in DRF, the logic inside the serializer's `update()` method prevents fields from being set to `None` or their default values unless they are explicitly sent in the request data.

#### 1\. HTTP PUT vs. DRF's Serializer

- **HTTP PUT Standard:** Yes, a pure RESTful PUT means you are **replacing** the entire resource. If you omit a field, the server should set it to its default/null value or whatever is appropriate for a completely new, incomplete object.

- **DRF `ModelSerializer` PUT:** DRF's default implementation is more forgiving. The base `ModelSerializer` logic iterates over the `validated_data` (which only contains the fields you sent, like `email`) and applies **only those fields** to the existing `instance`. Omitted fields are simply ignored and retain their existing values.

  The key logic that achieves this is usually in the `super().update(instance, validated_data)` call:

  ```python
  # Inside the ModelSerializer.update() method (simplified)
  for attr, value in validated_data.items():
      setattr(instance, attr, value)
  instance.save()
  return instance
  ```

  Since `validated_data` in your request only contained `"email"`, only the `email` attribute was updated on the `user` instance before saving. **`first_name`**, **`last_name`**, and other fields were never touched.

#### 2\. Why This DRF Behavior?

This behavior is a common **convenience abstraction** in DRF to make it easier for developers.

- **Practicality:** In many real-world APIs, truly replacing _every single field_ (including system-generated IDs, timestamps, or staff flags) is dangerous or impractical. Developers often treat `PUT` in DRF as a **"full update"** where all _`updatable`_ fields should be sent, but fields not sent are assumed to be left as-is, not wiped.
- **Preventing Accidental Deletion:** If omitting a field set it to `None`, a client bug or simple mistake could accidentally wipe out essential user data like their name or profile details.

---

### Question:

When sending a `PUT` request, I must include the email field, otherwise I get a 400 bad request error. Which part of code made this email field mandatory?

### Answer:

The part of your code that makes the `email` field **mandatory** when sending a `PUT` request is the default behavior of the **Django Rest Framework (DRF) `ModelSerializer`** combined with how you defined the `email` field in your **Django model**.

Here is the breakdown:

---

#### Code Analysis

#### 1\. Model Definition (`users/models.py`)

In your `User` model, the `email` field is defined without specifying it can be blank or null:

```python
class User(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True) # <--- This is the root cause
    # ... other fields ...
```

By default in Django, `models.EmailField()` (and `models.CharField()`, `models.TextField()`, etc.) are considered **required** unless you explicitly set `blank=True` or `null=True`. For database string fields like `EmailField`, the standard is to require a value.

#### 2\. Serializer Validation (`users/serializers.py`)

Your `UserSerializer` inherits from `serializers.ModelSerializer`.

```python
class UserSerializer(serializers.ModelSerializer):
    # ...
    class Meta:
        model = User
        fields = (
            "id",
            "email", # <--- Included here
            # ...
        )
```

During a `PUT` or `PATCH` request, DRF's `ModelSerializer` does the following:

1.  It identifies all fields included in the `Meta.fields` tuple (`"email"` in this case).
2.  It checks the underlying model definition. Since the `email` field in the model is **required** (as per point 1), the serializer treats it as a required field for **all update operations** (`PUT` and `PATCH`) by default.
3.  When you send a request body without the `email` field, the serializer's `is_valid()` method fails because a mandatory field is missing, resulting in the **400 Bad Request** error.

---

#### How to Change This Behavior

If you want to allow a `PUT` request that only updates, for example, the `first_name` without requiring the `email` (or any other field), you have two primary options:

#### 1\. Use PATCH for Partial Updates (Recommended)

The industry-standard approach is to use **`PATCH`** for partial updates where you only send the fields you want to change.

- When the request method is `PATCH`, you pass `partial=True` to the serializer instantiation:
  ```python
  # Inside your userProfile or userDetail view:
  # ...
  serializer = UserSerializer(user, data=request.data, partial=True)
  # ...
  ```
  Your `userDetail` view already implements this for `PATCH`: `serializer = UserSerializer(user, data=request.data, partial=True)`.

#### 2\. Override Serializer Field Requirement (If you insist on using PUT for partial updates)

If you must allow partial updates using the `PUT` method, you can explicitly define the `email` field in your serializer and mark it as optional:

```python
# users/serializers.py

class UserSerializer(serializers.ModelSerializer):
    # ...
    email = serializers.EmailField(
        required=False, # <--- Setting required=False overrides the model's requirement for updates
        unique=True
    )
    # ...
```

By setting `required=False` in the serializer, you tell DRF not to fail validation if the field is omitted during an update operation, resolving the 400 error.

---

### Question:

If the update method in the serializers doesn't care about the whole data being provided, so what is the point of this `partial=True` setting in
`UserSerializer(user, data=request.data, partial=True)`?

### Answer:

The short answer is that the `partial=True` setting is **not primarily for the `update()` method itself**, but for controlling the **validation phase** (`is_valid()`).

#### The Role of `partial=True`

The `partial=True` flag tells the serializer to ignore the `required=True` constraints for any field that is _omitted_ from the input data.

Here is the step-by-step process during an update request (`PUT` or `PATCH`):

#### 1\. Without `partial=True` (Default PUT)

When you omit the flag, DRF assumes a complete update.

- **Validation Phase (`is_valid()`):** The serializer checks all fields defined in the `Meta.fields` list. If a field is missing in the `request.data` **AND** its model definition (or explicit serializer definition) makes it mandatory (like `email` in your case), validation **fails** with a 400 Bad Request.
- **Purpose:** This enforces that if you're doing a `PUT`, you must provide a valid value for every required field, even if the value is the same as the current one.

#### 2\. With `partial=True` (Standard PATCH)

When you set `partial=True`, you explicitly signal that an incomplete set of data is acceptable.

- **Validation Phase (`is_valid()`):** The serializer relaxes its validation rules. It only validates the fields that are actually present in the `request.data`. All fields that are **omitted** are automatically treated as non-required.
- **Purpose:** This is essential for the `PATCH` method, allowing you to update a single field (e.g., just `first_name`) without the validation process failing because you didn't include `email` and `last_name`.

---

#### Why the `update()` Method Seems to Ignore It

You observed that your `PUT` request (which likely did **not** use `partial=True`) successfully updated only the `email` field. This is because the `update()` method is called **only after validation passes**, and it works like this:

1.  The `update()` method only iterates over the fields that made it through validation, which are contained in the `validated_data` dictionary.
2.  If you send: `{"email": "new@test.com"}`
    - **Validation (without `partial=True`):** Fails, because `first_name` and `last_name` are missing, and they are required.
    - **Validation (with `partial=True`):** Succeeds, because it ignores the requirement for the omitted fields.
    - **`update()` Method:** Is called with `validated_data = {"email": "new@test.com"}` and successfully applies only the email change.

**Crux:** The `partial=True` flag is crucial for allowing the validation to pass when data is incomplete. The underlying `update()` method is already flexible enough to only apply changes for the fields present in the input.

This is why, in function-based views, you must manually check the HTTP method and decide whether to pass `partial=True`:

```python
# The essential difference in your view logic:
partial_update = request.method == "PATCH" # True if PATCH, False if PUT

serializer = UserSerializerWithToken(
    user,
    data=request.data,
    partial=partial_update # <--- Controls the validation phase
)
```

---

### Question:

Is the `fields` option in `Meta` class of a serializer related to `serialization` (reading data) or `deserialization` (creating/updating data)?

### Answer:

The `fields` setting in the `Meta` class of a Django Rest Framework (DRF) `Serializer` is crucial because it relates to **both** serializing (output) and de-serializing (input/validation) data.

Here is a breakdown of how it affects both processes:

---

#### 1. Serialization (Output - Reading Data) ➡️

This is the process of converting complex Python objects (like your Django `User` model instance) into native Python datatypes (like dictionaries) that can be rendered into JSON/XML for the client.

- **What it Controls:** Only the fields listed in the `fields` tuple will be present in the final API response.
- **Example:** If your model has `email`, `password`, and `created_at`, but your `Meta` only lists `('email', 'created_at')`, the `password` field will not be included in the JSON output, protecting sensitive data.

---

#### 2. De-serialization (Input/Validation - Writing Data)

This is the process of converting incoming data (like JSON from a client) into complex Python objects for validation and saving to the database.

- **What it Controls:** The fields listed in the `fields` tuple are the only ones that the serializer will attempt to **validate** and process from the incoming `request.data`.
- **Mandatory Fields:** If a field is in `fields` and is considered **required** (either by your Django model definition or an explicit serializer field definition), validation will fail if that field is missing from the incoming request data (unless you use `partial=True`). This is the validation that blocked your incomplete `PUT` request for the `email` field.
- **Saving Data:** Only data for the fields listed in `fields` (and which pass validation) will be available in the `validated_data` dictionary used in the `.create()` and `.update()` methods.

---

#### Summary

The `fields` setting acts as a **gatekeeper** for your data.

| Process                      | `fields` Role                                                                    |
| :--------------------------- | :------------------------------------------------------------------------------- |
| **Output (Serialization)**   | Defines which fields are **exposed** to the client.                              |
| **Input (De-serialization)** | Defines which fields are **expected** from the client for validation and saving. |

By including a field in `fields`, you are telling DRF that the field should be used for **both** reading _and_ writing operations, unless you override its behavior (e.g., using `read_only=True` or `write_only=True`).

---

### Note:

The `write_only_fields` option on `ModelSerializer` has been deprecated and replaced with a more generic `extra_kwargs`.

```python
class MySerializer(serializer.ModelSerializer):
    class Meta:
        model = MyModel
        fields = ['id', 'email', 'password', 'is_admin']
        extra_kwargs = {
                'password': {'write_only': True},
                'is_admin': {'write_only': True}
        }
```

Alternatively, specify the field explicitly on the serializer class:

```python
class MySerializer(serializer.ModelSerializer):
    is_admin = serializers.BooleanField(write_only=True)

    class Meta:
        model = MyModel
        fields = ['id', 'email', 'notes', 'is_admin']
```

The `read_only_fields` option remains as a convenient shortcut for the more common case.

---

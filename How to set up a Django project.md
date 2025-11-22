# How to create a Django project:

## We need Python installed on our machine

So visit python.org and download and install the latest version or just install Anaconda. By installing Anaconda, we get Python and other most used Python libraries.
After that check python being installed by

```
py --version
```

## Create a folder:

Create a folder for our backend. Let's name it 'backend' and cd into it.

## Create a virtual environment

Use venv module, which is a part of Python's standard library, to create a virtual environment

```
py -m venv .venv
```

This command will create a '.venv' folder inside of the 'backend' folder.

The main external package that provides similar functionality is virtualenv.

---

#### Note: Python Version Management and Virtual Environments on Windows

#### Question

It seems common to have multiple Python versions installed on a Windows machine. For instance, I might install Python directly from the official python.org website and later install a distribution like Anaconda, resulting in two separate Python installations.

My first question is: What are the other typical ways one might end up with different Python versions on a single Windows system?

My second question is: When using the standard command to create a virtual environment, `py -m venv .venv`, which specific Python interpreter's version will be used for this new environment?

---

#### Answer

You've highlighted an excellent point. Managing different Python versions is a key challenge in development, and Windows users frequently encounter this.

##### 1. Additional Ways to Acquire Multiple Python Versions

The standard installer and distributions like Anaconda are the most common culprits, but several other mechanisms can lead to multiple Python versions on your machine:

- **Version Management Tools:** Dedicated tools such as **`pyenv-win`** are specifically designed to streamline the installation and management of various Python versions (e.g., 3.8, 3.10, 3.12). These tools create distinct, isolated installations, making it simple to switch between versions per project.
- **Specialized Development Environments:** Certain IDEs (like PyCharm or VS Code) or development utility suites may prompt you to install a project-local or necessary Python interpreter, contributing to your overall count of installed versions.
- **Windows Subsystem for Linux (WSL):** If you utilize WSL, the Python installation within your Linux distribution (e.g., Ubuntu) is entirely separate and distinct from any Python versions installed natively on the Windows operating system.
- **Third-Party Application Dependencies:** Various applications or system tools may silently install their own embedded Python interpreters to function, which are usually not intended for general user development.
- **Containerization Tools:** While not a direct system installation, tools like Docker often involve setting up containers based on specific Python versions, further demonstrating the need for version isolation.

##### 2. Determining the Interpreter for `py -m venv .venv`

The crucial factor in determining the version of a new virtual environment is **which specific Python interpreter is executing the `venv` module.**

In your example, `py -m venv .venv` uses the Python interpreter that the **Python Launcher for Windows (`py.exe`)** resolves to based on its selection logic.

**Selection Logic for `py` (without a version flag):**

1.  **Explicit Version Check (Overridden by Flag):** If you had run a command like `py -3.11 -m venv .venv`, the Launcher would have explicitly chosen the latest installed **Python 3.11** interpreter to execute the command.
2.  **Default Interpreter:** Since you did not provide a version flag, the `py` launcher defaults to the following priority:
    - It typically selects the **highest installed Python 3 version** it finds (e.g., if both 3.9 and 3.12 are present, it selects 3.12).
    - This behavior can be modified by setting the environment variable `PY_PYTHON` to specify a preferred default version.

**Conclusion:**

The new virtual environment, `.venv`, will inherit the exact version (and its standard libraries) of the Python interpreter that was selected by the **Python Launcher's default mechanism**.

**Best Practice:**

To eliminate all ambiguity and ensure project fidelity, it is best practice to explicitly instruct the launcher which version to use:

| Command                  | Resulting Python Version in `.venv`                                              |
| :----------------------- | :------------------------------------------------------------------------------- |
| `py -3.11 -m venv .venv` | **Guaranteed** to use the latest installed Python **3.11**.                      |
| `py -m venv .venv`       | **Relies on the system default** (usually the newest 3.x found by the launcher). |

---

## Activate virtual environment

For a Bash terminal on windows run

```
source .venv/Scripts/activate
```

to deactivate the virtual environment, just run

```
deactivate
```

Everything we need for our project must be installed in the virtual environment. So when installing a package for the project, remember to activate the virtual environment first.

## Install Django

```
pip install Django
```

To check Django installation, just simply run

```
py -m django --version
```

or

```
django-admin --version
```

another way is to open Python REPL by running

```
py
```

Then run

```
import django
django.get_version()
```

To quit REPL, run

```
quit()
```

## Create a Django project

Many people use the name 'config' for the 'Project Package' and many people have struggled with the same issue you went through. The Django team made a mistake with the below command many years ago and while it's a very confusing folder structure, they can't get out of it.

Note: When we create a Django project, Django writes some comments in some modules. In one of these modules the comments say something like "Hello from 'Project_Package_Name' project". So Django team considers the name you provide for project package to be your project's name, but obviously this can cause a lot of confusion. So people avoid it and many use 'config' name for project package.

```
django-admin startproject config .
```

Here 'config' is the name of " Project Package" (containing a settings.py and other files). It's the container for all of your project's main settings and configurations.
It's a directory that is the actual Python package for your project. Its name is the Python package name you’ll need to use to import anything inside it (e.g. config.urls).

Let's always use the name "config" for Project Package until we find a better one because many developers do this.

But in many tutorials, they use confusing names for it like 'my_project' or 'my_django_project'. It's because Django team seems to prefer this convention of naming project package as your project's name. But it's confusing because it's not the project folder, it's project package folder. Even in Django documentation, a name of 'mysite' has been used which is not good.

The next parameter in above command is there to specify "Project Directory" (By default contains manage.py and a project package).

By using '.' (a period), I'm telling Django to setup the project here in 'backend' folder and next to '.venv' folder.

Remember to put the '.' at the end or specify another location for Django project stuff. If not, Django will create a directory named as 'config' in 'backend' folder for all the Django stuff, so the same name as project package. So we'll end up having two nested folders with same name which is very confusing.

## Start the server:

Run:

```
py manage.py runserver
```

## Create an app in Django project

Stop the server by pressing Ctrl + C and run

```
py manage.py startapp posts
```

## Tell our Django project about this new app

Navigate to project package folder and open the settings.py and in INSTALLED_APPS list, add the new app name like this

```
INSTALLED_APPS = [
    ...
    'posts' ,
]
```

## Define a view

Navigate to views.py in new app folder and define a function view (there's also class views which I think is for advanced developers )

```python
from django.http import JsonResponse

def welcome(request):
    return JsonResponse("Hello world from /api/posts", safe=False)
```

## Create app's URL configuration file

To access the view in a browser, we need to map it to a URL - and for this we need to define a URL configuration, or `URLconf` for short. These URL configurations are defined inside each Django app, and they are Python files named `urls.py`.
In new app's folder, create a 'urls.py' file. This file is in charge of routing urls to views.

```python
from django.urls import path
from .views import welcome

urlpatterns = [path("", welcome, name="welcome")]
```

## Tell urls.py in project package about this file

The next step is to configure the root URLconf in project package to include the URLconf defined in posts/urls.py, so navigate to project package folder and open urls.py there. Add a path to urlPatterns.

The path() function expects at least two arguments: the first one is a route and the second one is a view or include() function. The include() function allows referencing other URLconfs.

Whenever Django encounters include(), it chops off whatever part of the URL matched up to that point and sends the remaining string to the included URLconf for further processing.

```
path('api/posts/', include('posts.urls'))
```

## Check the result in browser

Visit http://localhost:8000/api/posts/ and you should see the welcome message.

## Install Django Rest Framework

In project directory (backend folder) and with venv activated, run:

```
pip install djangorestframework
```

Visit https://www.django-rest-framework.org/#installation for more information.

## Add 'rest_framework' to your INSTALLED_APPS setting.

In project package, open up settings.py file and add 'rest_framework' in INSTALLED_APPS list:

```
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

## Create an API view using Django Rest Framework

Instead of 'welcome' view we defined before, do this:

```
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .posts import posts

@api_view(['GET'])
def getPosts(request):
    return Response(posts)
```

For more information, in django-rest-framework.org website, select 'Views' in 'API Guide' dropdown menu in navbar. Then from sidebar and under 'Function Based Views' select '@api_view()'.

## Handling CORS errors

If you're running your frontend app on port 3000 and your Django backend app on port 8000 and try to fetch data from backend for your frontend, you'll get CORS error. In a video the tutor explains how to handle it.

## Apply initial migrations

In project directory, run:

```
py manange.py migrate
```

## Creating a super user

Now that we've applied our migrations, we can create super users. In project directory, run:

```
py manage.py createsuperuser
```

It will prompt us to provide Username, Email and Password for this superuser.

Now you can visit Django admin panel in http://localhost:8000/admin, login with your Username and Password credentials and see database tables and manage them. Django admin panel is a good place to initially see database tables and manage them.

## Define database models

Navigate to your app's folder and open models.py file. This is where we define our models.

The first one is the User model and because it's created by Django itself, we just need to import it if we need to use it. We don't need to define it:

```
from django.contrib.auth.models import User
```

Define other models like:

```
from django.db import models

class Product(models.Model):
    ...
```

As you see a model is a class, we give it a name and as the argument, we provide 'models.Model'. By passing this argument, Django now knows this class is a model.

Then we define model's attributes.

About id attribute: By default, Django creates an 'id' attribute if there's no field set as primary key. If the frontend uses '\_id' instead of 'id', we can tell Django to use our '\_id' attribute as primary key like this:

```
_id = models.AutoField(primary_key=True, editable=False)
```

So now there won't be an automatically generated 'id' field for this model.

In above code, we're telling Django that the '\_id' attribute needs to be:

- filled automatically by Django
- used as primary key
- uneditable

(The tutor said 'User' model won't have '\_id' attribute because this model was created by Django and said I don't want to change it)

The first attribute is 'user',

```
class Product(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
```

The 'user' is one of the staff in company who creats this product, a 'user' can create many products, so it is a one-to-many relation-ship. In Django, a one-to-many relation-ship is implemented using 'ForeignKey' field. We've provided the 'User' model we imported as the first argument.

As the second argument we're telling Django that if that user is deleted (after leaving their job or getting fired), we don't want to delete all the products they created, but just set user attribut in those products to 'null'.

### Note:

I've refined the text to be clearer, simpler, and more natural while keeping the technical meaning correct.

---

#### 📝 What `SET_NULL` Means for Django Models

**Question 1:** In this Django model, what is `SET_NULL`?

```python
class Product(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
```

**Answer 1:**

`SET_NULL` is a simple **rule** that tells the database what to do when the user who owns a product is deleted.

If the **`User`** is deleted, the `user` field on all of their products will be **cleared** and set to **empty (`NULL`)**. This means the product stays in your database, but the link to the original user is gone.

You must also set **`null=True`** on the field. This tells the database that it's okay for this column to be empty.

---

**Question 2:** Is `SET_NULL` a method or just a value for `on_delete`?

**Answer 2:**

`SET_NULL` is **just a value** (a constant). It is not a function or method that you call.

You pass this special **value** to the `on_delete` setting:

```python
on_delete=models.SET_NULL
```

Django sees this value and creates a database rule (a foreign key constraint) that handles the clearing automatically when a linked record is deleted.

---

### Note: Python's `None` vs. SQL's `NULL`

In Python, we **don't have "null"** like in JavaScript or C++. Instead, the concept for a missing or empty value is represented by the constant **`None`**. It's an object of its own data type, called `NoneType`.

#### The Database Connection

You see the word `null` in Django settings (like `null=True` or `models.SET_NULL`) because Django is interacting with an **SQL database**.

- In SQL databases (PostgreSQL, MySQL, etc.), the term for a missing value in a column is **`NULL`**.

#### Translation Between Systems

When your Django application fetches data from the database, a translation happens:

1.  **Database $\rightarrow$ Python:** Any database column that has a **`NULL`** value is automatically converted and represented in your Python code as the value **`None`**.
2.  **Purpose of Settings:** The `null=True` or `models.SET_NULL` settings are just Django's way of telling the SQL database that a specific column is allowed to contain that database-level **`NULL`**.

**In short:** Python uses **`None`**, but Django uses the SQL-based term **`NULL`** when defining the structure and behavior of your data fields.

---

Here is the whole 'Product' model:

```
from django.db import models
from django.contrib.auth.models import User

# Create your models here.


class Product(models.Model):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    name = models.CharField(max_length=200, null=True, blank=True)
    image = models.ImageField(null=True, blank=True,
                              default='/placeholder.png')
    brand = models.CharField(max_length=200, null=True, blank=True)
    category = models.CharField(max_length=200, null=True, blank=True)
    description = models.TextField(null=True, blank=True)
    rating = models.DecimalField(
        max_digits=7, decimal_places=2, null=True, blank=True)
    numReviews = models.IntegerField(null=True, blank=True, default=0)
    price = models.DecimalField(
        max_digits=7, decimal_places=2, null=True, blank=True)
    countInStock = models.IntegerField(null=True, blank=True, default=0)
    createdAt = models.DateTimeField(auto_now_add=True)
    _id = models.AutoField(primary_key=True, editable=False)

    def __str__(self):
        return self.name
```

Without the following method,

```
def __str__(self):
        return self.name
```

when we visit admin panel, in products list, we won't see each product by its name, so we add this method.

### Note:

That's right that adding the `__str__` method is crucial for representing data properly in the Admin Panel, but it's useful in several other places too.

The `__str__` method is a standard Python "dunder" (double underscore) method that defines the **informal, human-readable string representation** of an object. Any time Django or Python needs to display your model instance as text for a general user, it calls this method.

---

### Uses of `__str__` Beyond the Admin Panel

The main benefit is improved **readability** and **debugging** across your entire application:

#### 1. Relationships (Foreign Keys)

When you reference this `Product` model from another model using a `ForeignKey`, Django automatically uses the `__str__` output in its own code.

- **Example:** If you had an `OrderItem` model that links to `Product`, the Django logs, debug output, and query results showing that link would use the product's **name** (e.g., "AirPods Pro") instead of a generic ID (e.g., `<Product object (5)>`). This makes reading query results much faster.

#### 2. Template Display

If you pass a `Product` object directly to a Django template and try to print it, the `__str__` method is called.

- **Example:** In a template, if you write `{{ product }}`, it will output the product's name (e.g., "iPhone 15") because of your `__str__` method. Without it, you'd just see a non-helpful string like `<Product object (12)>`.

#### 3. Debugging and Logging

When you print an object in your Python code (views, managers, etc.) or log it, the `__str__` method is used.

- **Example:** If you're debugging a view and write `print(product_instance)`, the console output will clearly show the product's name, which is far more useful than seeing only the object's class and memory address.

#### 4. Custom Forms and Utilities

Any third-party package or custom utility code in your project that needs to display a product's identity will rely on the `__str__` method to get a meaningful name.

In short, it's a **best practice** to always define `__str__` because it ensures all parts of your application and development tools display your objects in a human-friendly way.

---

### Note:

When creating a model in models.py file and setting up a relationship (like a ForeignKey from a Product to a User or from a Comment to a Blog), the order in which models are defined can matter if you don't use a string for the related model's name.

The problem arises when you try to reference a model that hasn't been defined yet in the file.

However, if you use the model's name as a string literal (e.g., models.ForeignKey('OtherModel')), the order of definition becomes completely irrelevant. Because using a string is a simple fix and the best practice, model order is rarely a practical issue in real-world Django development.

## Install pillow library

To use `models.ImageField`, we need to install `Pillow` which is an image processing library. So while in venv, run:

```
pip install pillow
```

---

### Note:

Installing Python packages with `pip` is generally **case-insensitive** for the package name itself. So, using **`pip install Pillow`** will typically work just as well as **`pip install pillow`**.

Here's why and the important thing to remember:

### 1\. `pip` Installation is Flexible

When you use `pip install`, the command searches the `PyPI` (Python Package Index) repository. PyPI's search mechanism is designed to be case-insensitive to help users find the correct package even if they don't know the exact capitalization.

### 2\. Python Imports are Strict

While installation is flexible, your Python code must be **case-sensitive** when you actually import the library.

- **Correct Code:** `from PIL import Image` (Pillow is imported as `PIL` by convention and design).
- **Incorrect Code:** `from pil import Image` (This will cause an error because Python can't find the `pil` module).

### 3\. Best Practice

Even though the installation is flexible, the best practice is to use the **lowercase spelling** for installation because it matches the convention used in your project's `requirements.txt` file and is the official package name on PyPI (which is usually all lowercase):

```bash
# Best practice for installation:
pip install pillow

# Correct for your requirements.txt:
pillow==10.1.0
```

But in the case of `Pillow`, running `pip install Pillow` will still successfully install the `pillow` package.

---

### Note:

The `models.ImageField` is Built-In, but Unusable without `Pillow`. The model field `models.ImageField` is included in Django's code base from the start; you don't install a separate module to get it into django.db.models. However, the field is designed to perform image-specific operations that require `Pillow` as an external image processing library:

- Validation: It needs to confirm that an uploaded file is actually a valid image (JPEG, PNG, etc.), not just a file renamed with an image extension.
- Metadata: It needs to check and save the image's height and width into its attributes. Since ImageField cannot perform these core functions without Pillow (the modern fork of PIL), Django enforces the dependency.
- What Happens Without Pillow? If you use `models.ImageField` in your model but haven't run `pip install pillow`, Django's system checks will fail (usually during makemigrations or when the application starts). You'll receive a clear error message like: `Cannot use ImageField because Pillow is not installed`. So, the field exists in the code, but it is functionally disabled until its necessary dependency is installed.

---

Question:

So we can say that the Django library is dependant on the pillow library in some way, right?

Answer:

Yes, you can absolutely say that a Django project using `models.ImageField` is **dependent** on the **Pillow** library.

The dependency works like this:

1.  **Django Code:** The field class, `models.ImageField`, exists within Django's core code.
2.  **Required Functionality:** The code within `models.ImageField` contains instructions to call image processing functions (like checking format or size).
3.  **Pillow:** The actual image processing functions are **not** in Django; they are provided by the **Pillow** library.

Therefore, for the `ImageField` to be functional, Pillow must be installed in your environment. If you remove Pillow, any part of your Django project relying on that field will stop working, demonstrating a clear dependency.

---

Question:
For the image field, can we use other library than pillow?

Answer:
No, for Django's built-in `models.ImageField`, you cannot use a library other than Pillow.

The code for ImageField is hardcoded within Django to rely specifically on Pillow's functions for image validation and metadata retrieval. If you need a different library (like OpenCV), you must use the generic `models.FileField` and handle the image processing yourself in custom Python code.

---

## Make and Apply migrations

Now that we've changed our models, we need to make migrations again:

```
py manage.py makemigrations
```

By running this command, a file named '0001_initial.py' will be added to 'migrations' folder in app's directory.

Then we need to apply the migrations. Run:

```
py manage.py migrate
```

Everytime we make changes to our models, we need to do these steps.

## Register your model to admin panel

Open admin.py file in your app's directory and add this code:

```
from django.contrib import admin
from .models import *
# Register your models here.

admin.site.register(Product)
# admin.site.register(Review)
# admin.site.register(Order)
# admin.site.register(OrderItem)
# admin.site.register(ShippingAddress)
```

After adding all the models to this file, our admin.py file content would be something like above.

Now if we visit admin panel, we will see our app and our 'Products' table there and we can add a product.

## Visit admin panel

We've done these steps by now:

- Defined our models
- Made migrations
- Applied migrations
- Registered our models to admin panel

So now we can visit admin panel and create and manage instances of our models.

## About image in product

## Using admin panel to add an image to a product.

 <!-- //Todo: Next we need to learn how to use frontend and backend logic to add an image to a product  -->

We can use Django admin panel to create a product and specify an image to it. By default, Django would store the image file in project directory (in the same level as manage.py) because Django would use the directory defined by your `MEDIA_ROOT` setting which by default points to project directory.

---

### Note:

When you specify the MEDIA_ROOT setting in your Django project, all user-uploaded files, regardless of their type (images, PDFs, videos, documents, etc.), will be stored within that base directory.

The term "Media" in `MEDIA_ROOT` is just Django's conventional term for user-uploaded content, distinguishing it from "Static" files (which are assets created by the developer).

So `MEDIA_ROOT` is not just about media, meaning images and videos, but all user-uploaded files.

And when we say "user-uploaded" we mean "admin-uploaded" too!

---

### Note:

Here is what you need to know about where files are stored and how to control the specific folder structure:

#### 1\. How Django Determines Storage Location

Django uses two settings to manage user-uploaded files (known as "media files"):

1.  **`MEDIA_ROOT` (The Base Directory):** This is the **absolute file-system path** to the directory where all user-uploaded files will be placed. You must define this in your `settings.py`.
    - **Example:** If you set `MEDIA_ROOT = BASE_DIR / 'media'`, all uploads go into the `media` folder at your project root.
2.  **`upload_to` (The Subdirectory):** This is the parameter you set on the model field itself, which defines the **subdirectory** _within_ `MEDIA_ROOT` where the file should go.

---

#### 2\. Storing Images in a Different Folder

To store images in a dedicated folder (e.g., inside an `product_images` folder, within your main `MEDIA_ROOT`), you only need to adjust the **`upload_to`** parameter in your model.

In your `Product` model, you need to modify the `image` field:

#### The Correct Way to Store Images

```python
class Product(models.Model):
    # ... other fields ...

    # 🛑 The current setting stores files directly in MEDIA_ROOT
    # image = models.ImageField(null=True, blank=True, default='/placeholder.png')

    # ✅ The new setting creates a 'product_images' folder inside MEDIA_ROOT
    image = models.ImageField(
        null=True,
        blank=True,
        default='/placeholder.png',
        # Set the subdirectory here:
        upload_to='product_images/'
    )

    # ... rest of the model ...
```

#### The Resulting Path

If your settings are:

- `MEDIA_ROOT = /path/to/your/project/media/`
- `upload_to='product_images/'`

And you upload a file named `widget.jpg`, the file will be saved to:

`/path/to/your/project/media/product_images/widget.jpg`

By only adjusting the `upload_to` parameter, you gain complete control over the folder structure for your uploaded files without having to change your primary `MEDIA_ROOT` setting.

---

So we create two folders at the root directory of our Django project: a `media` folder and a `frontend` folder. As their name implies, we use media folder for our user-uploaded files and frontend folder for our React frontend files.

<!-- Todo: Later when we add frontend stuff in this folder, we need to configure the STATICFILES_DIRS setting and tell Django about frontend folder like this:
STATICFILES_DIRS = [
     BASE_DIR / 'frontend/build/static'
]    -->

But there's one more problem. When we click on an image in admin panel to see the image, the url is not okay so we need to fix this.

In settings.py and below 'STATIC_URL = 'static/'
' add MEDIA_URL:

```
MEDIA_URL = '/media/'
```

---

### Note: Media File URL Construction

The public web URL for any user-uploaded file is created by **concatenating** the `MEDIA_URL` setting with the `upload_to` parameter defined on the model field.

- **Public URL** = **`MEDIA_URL`** + **`upload_to`** + **`filename.ext`**

#### Example:

If you set:

- `MEDIA_URL = '/media/'`
- Your model has: `image = models.ImageField(upload_to='product_images/')`
- The uploaded file is: `widget.jpg`

The final URL will be:

`http://yourdomain.com/media/product_images/widget.jpg`

---

Then in project package, open urls.py file and add these imports:

```
from django.conf import settings
from django.conf.urls.static import static
```

and then, after 'urlpatterns' list, add this line:

```python
# CRITICAL: ONLY for Development (DEBUG=True)
if settings.DEBUG:
    # Adds URL patterns to serve files from MEDIA_ROOT
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

### Note:

The above code is **adding a set of URL patterns** to your main `urlpatterns` list, specifically enabling Django's development server to serve **user-uploaded media files**.

#### 1. What `+=` Is Doing

The `+=` operator is being used to **append (add)** the items returned by the `static()` function to the existing `urlpatterns` list.

- In Python, for lists, the `+=` operator is equivalent to using the `.extend()` method. It takes all the items from the iterable on the right (the patterns returned by `static()`) and adds them individually to the end of the list on the left (`urlpatterns`).

- **Result:** `urlpatterns` starts as a list of your app's paths, and the `static()` function generates a new list of special paths required for media files, which are then appended to `urlpatterns`.

---

#### 2. What the Full Line is Doing

The entire line, `urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)`, performs a **critical setup step for media files in a development environment (`DEBUG=True`)**.

It tells Django:

| Parameter            | Value                 | Purpose                                                                                                                                |
| :------------------- | :-------------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| `settings.MEDIA_URL` | e.g., `/media/`       | Defines the **URL prefix** that the server should listen for (e.g., any URL starting with `/media/`).                                  |
| `document_root`      | `settings.MEDIA_ROOT` | Defines the **physical directory** on the file system where the server should look for the files requested via the `MEDIA_URL` prefix. |

**In simple terms, it activates a temporary URL handler.** This handler ensures that when the browser requests an image at a URL like `/media/product_photos/widget.jpg`, Django's development server knows to look for that file inside the folder specified by `MEDIA_ROOT`.

**Crucial Note:** This code block is **only** placed inside an `if settings.DEBUG:` check because you **must remove it** in production. In a production environment, a dedicated web server (like Nginx or Apache) handles serving media files directly, bypassing Django entirely.

---

### Note:

The function call **`static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)`** returns a **list of one or more URL pattern objects**, not a single item.

When you use the `+=` operator:

```python
urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

1.  The `static()` function generates all the necessary URL pattern objects needed to handle requests for media files (like patterns for the file itself, and potentially patterns for directory listings if enabled).
2.  The `+=` (or `extend`) operator then adds **all** these generated patterns to the existing `urlpatterns` list.

So, while it looks like a single function call, it's adding multiple pieces of plumbing to your URL configuration.

---

## Serializing Data

To actually return data from database, open views.py file in products app. Then import 'Product' model:

```
from .models import Product
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET'])
def getProducts(request):
    products = Product.objects.all()
    return Response(products)
```

But we get an error here because we're working with Django Rest Framework and we need to serialize our data before returning it back to our frontend.

We need to create a serializer for every model we want to return. A serializer is going to wrap our model and turn that into JSON format.

You can't return data like that because Django querysets (like `products = Product.objects.all()`) are made up of complex **Python objects** (model instances), and the client (the browser or React app) expects data in a standard format like **JSON** (JavaScript Object Notation).

**Serializers** are required to perform the crucial step of **translation**: converting those complex Django objects into native Python data types (like dictionaries and lists) that can then be easily rendered into JSON for transmission over the network. Returning the raw queryset object directly will cause an error.

So in app folder, create a file named 'serializers.py' and put this code inside:

```
from rest_framework import serializers
from .models import Product

class ProductSerializer(serializers.ModelSerializer):
    class Meta:
        model = Product
        fields = '__all__'
```

Then navigate back to views.py file in app's folder and import this serializer and use it like this:

```
from .models import Product
from rest_framework.decorators import api_view
from rest_framework.response import Response
from .serializers import ProductSerializer

@api_view(['GET'])
def getProducts(request):
    products = Product.objects.all()
    serializer = ProductSerializer(products, many=True)
    return Response(serializer.data)

```

By 'many=True' argument we are telling Django that we want to serialize many objects. In other words, we have a list of objects we want to return not just one object.

In 'getProduct' view, we'll set it to False because there, we just want to return one object. Update 'getProduct' view like this:

```
@api_view(['GET'])
def getProduct(request, pk):
    product = Product.objects.get(_id=pk)
    serializer = ProductSerializer(product, many=False)
    return Response(serializer.data)
```

---

### Note: more about serializers

#### 1\. Serializers are Bidirectional (Serialization & Deserialization) 🔄

Django REST Framework (DRF) serializers are powerful because they are **bidirectional**. This means the same `ModelSerializer` definition is used for two distinct processes:

- **Serialization (Reading/Output):** Converting complex Django model instances (Python objects) into simple, standardized data formats, typically **JSON**, for sending data _out_ to the client (e.g., handling a `GET` request).
- **Deserialization (Writing/Input):** Taking raw data received from the client (e.g., a `POST` or `PUT` request), validating it against the model's rules, and converting it back into a Python object ready to be saved _into_ the database.

When handling a write request, you initialize the serializer with the incoming data (`serializer = ProductSerializer(data=request.data)`) and, after validation, call `serializer.save()` to trigger the database operation and any associated file uploads.

---

#### 2\. Direct Model Assignment vs. Best Practice API

Your tutor's use of **direct model assignment** (e.g., `product.name = data["name"]; product.save()`) to create or update data is generally **not the best practice** for APIs built with Django REST Framework.

While this approach keeps initial tutorials simple by focusing on direct model manipulation, it has severe limitations in a robust API:

- **Bypasses Validation:** It skips `serializer.is_valid()`, meaning any invalid data (e.g., a string for a number field) will cause a server error instead of a clean `HTTP 400 Bad Request` response.
- **Poor Error Handling:** Errors are handled by low-level database exceptions instead of professional, API-friendly error messages provided by the serializer.

The recommended best practice is to always use the serializer for both creation and updates, as it centralizes validation, error handling, and file management.

---

#### 3\. React/Axios POST Request with Image and Django FBV Example 🛠️

Here is an example showing the necessary code flow from the React client to the Django Function-Based View (FBV) for creating a product with an image.

#### Frontend Logic (React/Axios)

The key is using the browser's **`FormData`** object to correctly package binary data (the image file) for the request.

```javascript
// Example component function in React (simplified)
import axios from "axios";

const uploadProduct = async (productData, imageFile) => {
  // 1. Create the FormData container
  const formData = new FormData();

  // 2. Append all text fields
  formData.append("name", productData.name);
  formData.append("brand", productData.brand);

  // 3. Append the image file
  // 'image' must match the field name in your Product model/serializer
  formData.append("image", imageFile);

  // 4. Send the POST request
  try {
    const config = {
      headers: {
        "Content-Type": "multipart/form-data", // The browser handles this boundary
        // Include authorization token if needed
        Authorization: "Bearer YOUR_JWT_TOKEN",
      },
    };

    const response = await axios.post(
      "/api/products/create/",
      formData,
      config
    );
    console.log("Product created:", response.data);
  } catch (error) {
    console.error("Error uploading product:", error);
  }
};
```

#### Backend Logic (Django Function-Based View)

The FBV uses the serializer to seamlessly handle the deserialization (validation and saving) of both the text fields and the image file.

```python
# views.py (Django REST Framework)

from rest_framework.decorators import api_view, permission_classes
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework import status
# ... import Product, ProductSerializer ...

@api_view(['POST'])
@permission_classes([IsAuthenticated])
def create_product(request):
    # 1. Instantiate the Serializer with incoming data
    # request.data intelligently holds all the FormData content
    serializer = ProductSerializer(data=request.data)

    # 2. Validate the data (including the image file)
    if serializer.is_valid():

        # 3. Save the new model instance
        # The serializer automatically saves the image to MEDIA_ROOT
        # and saves the path to the database.
        serializer.save(user=request.user)

        # 4. Return success response
        return Response(serializer.data, status=status.HTTP_201_CREATED)

    # 5. Return error response
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

---

### Note:

#### Why `ProductSerializer(data=request.data)` is Used

**`ProductSerializer` is a class**, not a function. When you use the syntax `ProductSerializer(...)`, you are not calling a function; you are **creating an instance of that class**.

#### 1\. The Class Constructor (`__init__`)

When you create an instance of any class in Python, the code automatically calls a special method within that class called the **constructor**, which is always named `__init__`.

When you write:

```python
serializer = ProductSerializer(data=request.data)
```

The code is actually doing this behind the scenes:

```python
serializer = ProductSerializer.__init__(data=request.data, *args, **kwargs)
```

The `__init__` method inside the `ModelSerializer` class is designed to accept different keyword arguments (like `instance` for reading, or `data` for writing) to set up the serializer's initial state. The serializer stores the data you pass in so it can be processed later when you call `.is_valid()` and `.save()`.

#### 2\. Why Use `data=`?

The `data=` syntax is used because the `__init__` method of the `ModelSerializer` accepts the keyword argument named `data`.

- **DRF's Convention:** The DRF serializer's `__init__` method is designed to be called with:
  - **`instance=`**: If you are **reading** data or **updating** an existing object.
  - **`data=`**: If you are **writing** a new object or updating an existing object.

---

#### The Equal Sign (`=`) in Python Arguments

When you want to pass data to a function or a class constructor (the `__init__` method), using the equal sign (`=`) is how you specify a **keyword argument**.

In Python, there are two main ways to pass arguments:

| Argument Type  | Syntax                            | Purpose                                                            |
| :------------- | :-------------------------------- | :----------------------------------------------------------------- |
| **Positional** | `function(value1, value2)`        | Values are matched to parameters based on their **order**.         |
| **Keyword**    | `function(name=value, age=value)` | Values are matched to parameters based on their **explicit name**. |

In your serializer example, you are using a **keyword argument**:

```python
ProductSerializer(data=request.data)
# Here, you are explicitly saying: "Assign the value of request.data
# to the parameter named 'data' in the __init__ method."
```

Using keyword arguments is generally considered best practice, especially with complex functions or constructors like those in DRF, because it makes the code clear and the order of arguments doesn't matter.

---

### Note:

When the tutor serialized the product for return, they likely used this syntax:

```python
# Assuming 'product' is a single model instance
serializer = ProductSerializer(product)
# OR: serializer = ProductSerializer(products, many=True)
```

And yet, I stated that the keyword should be `instance=`.

#### Why the Keyword `instance=` is Optional

When you call the serializer, the argument that represents the object(s) you are reading or updating is the **first argument**, and it is considered a **positional argument**.

1.  **If you pass it by position (no keyword):**

    ```python
    serializer = ProductSerializer(product)
    ```

    Python and DRF are smart enough to recognize that since this is the _first argument_ and you didn't pass `data=` for writing, this object must be the **instance** that needs to be read (serialized).

2.  **If you pass it by keyword (safer):**

    ```python
    serializer = ProductSerializer(instance=product)
    ```

    This works exactly the same way, but it is often clearer because it explicitly labels the object as the `instance`.

### The Rule of Thumb in DRF

The keyword argument is typically only **required** when you are passing **both** the existing object **and** the new data for an update (`PUT` or `PATCH`):

```python
# When updating, you MUST use the keywords to distinguish the two arguments
serializer = ProductSerializer(instance=product_to_update, data=request.data)
```

In your tutor's case (just reading/returning data), they could skip the `instance=` keyword because it was the only object passed, and DRF automatically interpreted it as the instance to be serialized.

---

## Authentication and Authorization

Because we're using DRF, handling authentication and authorization is a bit different than using Django built-in authentication and authorization system.

You can read more about authentication in DRF in their website. From 'API Guide' dropdown menu, click on 'Authentication' and specially search for token-based authentication.

In this project, we want to use token-based authentication using a third party package called 'Simple JWT'.

Simple JWT is a JSON Web Token authentication plugin for the Django REST Framework.

For full documentation, visit https://django-rest-framework-simplejwt.readthedocs.io/en/latest/.

To install the package, cd into project directory and run:

```
pip install djangorestframework-simplejwt
```

### Project Configuration

Then, your django project must be configured to use the library. In settings.py, add rest_framework_simplejwt.authentication.JWTAuthentication to the tuple of authentication classes:

```
# This is for Django project to be configured to use 'Simple JWT' auth library.
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
}
```

copy and past the above code under 'INSTALLED_APPS'.

### Create a `users` app

So far the only app we had was `products` app. Create a `users` app so we can move all the users related logic there, including Simple JWT logic.

In your root urls.py file, include the `users` app url config file like this:

```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/users/',include('users.urls')),
    path('api/products/',include('products.urls'))
]
```

Then in users app url config file, include routes for Simple JWT’s TokenObtainPairView and TokenRefreshView views (these two are class-based views):

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    # TokenRefreshView,
)

urlpatterns = [
    ...
    path('login/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    # path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

The tutor said that he doesn't want to use refresh tokens, instead he wants to extend the access token's lifespan to 1 month so he doesn't need refresh tokens. So I commented out two lines from above.

Note: This is just for convenience. In real projects we should consider using refresh tokens too, because it's more secure.

Note: The documentation uses the paths `api/token/` and `api/token/refresh/` for the views, but tutor used `api/users/login/` and I used it too:

```python
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
)

urlpatterns = [
    ...
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    # path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

Now visit this path in the browser. We'll see an HTML page with some data and a Login form. So where this page and form has come from?

It is generated because of DRF Browsable API feature.

The Django REST Framework (DRF) Browsable API is a feature that automatically generates a user-friendly HTML page when you view your API endpoints in a web browser.

In simple terms, it's a developer tool that turns your raw, hard-to-read JSON or XML API responses into an interactive web page that you can test and explore.

DRF endpoints are browsable: DRF's default configuration makes API endpoints accessible and interactive when viewed in a web browser, unlike most standard REST APIs that only return raw data.

Interaction: This interface isn't just for viewing data; it includes auto-generated HTML forms for endpoints that accept input (like POST or PUT), allowing developers to interact with and test the API without relying on external tools like Postman, Insomnia, or custom client-side code.

It is a core design philosophy of DRF to make API development and testing as easy and visual as possible.

### When DRF Returns HTML (The Browsable API)

DRF returns the HTML page (the Browsable API) only when both of these conditions are met:

- The Client is a Web Browser: The incoming request includes an HTTP header (specifically the Accept header) indicating that the client prefers or accepts HTML (text/html). When you type a URL into a browser, the browser sends this header by default.

- The HTMLRenderer is Active: The endpoint you are hitting must have the HTMLRenderer enabled in its configuration (which is DRF's default setting). This renderer is responsible for converting the API's response data into the interactive HTML format.

Example Scenario
You access the URL in your browser: Browser sends Accept: text/html. → DRF returns HTML.

### When DRF Returns Data (The Standard API Response)

DRF will return the standard raw data format (usually JSON) when either of the following conditions is met:

- The Client Explicitly Requests JSON: The client sends an Accept header that explicitly requests JSON (application/json) and does not list HTML as a preference. This is the behavior of virtually all non-browser clients.

Example: A mobile app, a React/Vue frontend fetching data via fetch or Axios, or a tool like Postman.

- The HTMLRenderer is Disabled: You, the developer, can explicitly remove the HTMLRenderer from the list of renderers for a specific view or globally in your settings.py. This forces the endpoint to only serve JSON, regardless of what the client requests.

Example Scenario
You send a request from Postman: Postman sends Accept: application/json. → DRF returns raw JSON.

Your frontend sends an AJAX request: Frontend sends Accept: application/json. → DRF returns raw JSON.

In short, DRF prioritizes content negotiation. It returns the format the client asks for, and the HTML Browsable API is simply the user-friendly format it provides when a browser comes calling.

## Simple JWT Authentication and Customization

### 1\. Generating the Initial Token

The primary purpose of the `/api/users/login/` route is to authenticate a user and generate a JWT pair.

If I submit my user credentials (username and password) to this endpoint and hit **POST**, the system handles the authentication. If successful, it returns an **access token** and a **refresh token**. This is the token we will store on the client-side (e.g., in Local Storage) and use to authenticate subsequent requests to protected API routes.

Let's test this with the user "Dennis". After submitting the credentials and hitting **POST**, we receive the tokens:

```json
{
  "refresh": "...",
  "access": "..."
}
```

### 2\. Debugging the Token on jwt.io

To understand the token's contents, we can copy the generated **access token** and paste it into a debugger like [jwt.io](http://jwt.io).

By debugging the token, we can see the Decoded Header:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

and the Decoded Payload:

```json
{
  "token_type": "access",
  "exp": 1759734793,
  "iat": 1759734493,
  "jti": "4e07512ec56b46c3a3e76329dc515270",
  "user_id": "1"
}
```

- **alg:** It uses `HS256` algorithm for signing.
- **token_type:** It's an `access` token.
- **exp:** It shows the precise time the token will expire (e.g. Mon Oct 06 2025; you can see this by hovering on exp value in `jwt.io` ).
- **user_id:** Crucially, it contains the primary key (user_id) of the user from the Django database.

### 3\. Demonstrating user_id Encoding

To confirm how the user_id is encoded, I'll demonstrate with a new user:

1.  I'll log into the **Django Admin Panel** and create a new user named "Tim" with the ID of **2**. I'll also add an email address (`tim@email.com`).
2.  I'll generate a new token using "Tim's" credentials.
3.  When I copy this new token and paste it into **jwt.io**, the decoded payload now clearly shows the `user_id` is **2**.

This confirms that by default, the token payload only contains the **user_id**, which the server uses to identify the authenticated user.

---

## Customizing Token Settings

Here we want to customize the token's lifetime and its contents (claims).

### 1\. Customizing Token Expiration

By default, the access token is set to expire after 5 minutes. We can change this by modifying the `SIMPLE_JWT` settings in `settings.py` in project package.

Visit DRF Simple JWT website and `Settings` section using this [link](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/settings.html). There we can see a Python dictionary named `SIMPLE_JWT` that shows all the attributes we can customize to change Simple JWT’s behavior.

You don't have to copy all the other default settings you don't want to change. You only need to define the SIMPLE_JWT dictionary in your settings.py file and include only the keys you want to override.

The default settings show an access token lifetime of 5 minutes and a refresh token lifetime of one day.

To make the access token last longer, we modify the settings in `settings.py` in project package. So just underneath REST_FRAMEWORK, past the code:

```python
from datetime import timedelta

SIMPLE_JWT = {
    # ... other default settings ...
    'ACCESS_TOKEN_LIFETIME': timedelta(days=30),  # Changed from minutes=5
    # The refresh token lifetime will remain at the default (1 day)
}
```

Note: The imports don't have to be at the top of files.

**Testing the Change:**

1.  After generating a new token (using the same `POST` request), the previously generated token's expiration date will remain the same.
2.  However, the new token we just generated will now reflect the change.
3.  When pasting the new token into **jwt.io**, the expiration date will be set for **30 days** from now, confirming the customization.

### 2\. Customizing Token Claims and API response

### Note:

The term `claim` is a specific technical term used almost exclusively within the context of JSON Web Tokens (JWTs) and related identity and security standards (like OAuth 2.0 and OpenID Connect).
A key-value pair in the JSON object that forms the JWT payload is called a "claim." Note that not any key-value pair in JS object or JSON object is called a claim but one in a JWT's payload .

Currently, our token payload only contains the user ID. For the frontend or other services, it's often useful to include more information, such as the user's **username**, **first name**, or **email**, directly inside the token.

We also want to add more data than `access` and `refresh` tokens to the API response.

The next step will be to explore the documentation for **"Customizing Token Claims"**, implementing a way to encode additional user data into the JWT payload, allowing us to decode and use that extra information.

**Customizing Simple JWT: Claims and Response Data**

In this section, we will tackle two critical customizations for Simple JWT:

1.  **Encoding additional data (claims)** into the JWT itself.
2.  **Customizing the API response** to immediately return user data alongside the tokens.

### 1\. Encoding Custom Claims into the JWT

By default, the Simple JWT only encodes the user_id and standard claims (like expiration time). We can add custom data. Visit DRF Simple JWT website and `Customizing token claims` section using this [link](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/customizing_token_claims.html).

**From SimpleJWT Documentation:**

If you wish to customize the claims contained in web tokens which are generated by the TokenObtainPairView and TokenObtainSlidingView views, create a subclass for the desired view as well as a subclass for its corresponding serializer. Here’s an example of how to customize the claims in tokens generated by the TokenObtainPairView.

To achieve encoding custom claims into the JWT, we have two options.

## Two Options for Customizing Simple JWT Claims

### Option 1: Configuration Override (Newer Documentation)

This method involves a little less code and relies on Django's configuration system.

- **Implementation:** Create a **custom serializer** (inheriting from `TokenObtainPairSerializer`) where you define your custom claims in the `get_token` method. Then, you reference this custom serializer in your `settings.py` file using the **`TOKEN_OBTAIN_SERIALIZER`** setting.
- **Result:** The custom serializer completely replaces the default one for the token generation endpoint.
- **Key Detail:** As noted in the documentation, claims added this way will be present in **both the Access Token and the Refresh Token**, since the `get_token` method is involved in generating both.

---

### Option 2: Custom View Inheritance (Older Documentation/Best Practice)

This method requires more explicit code but offers greater control over the process.

- **Implementation:** Create a **custom serializer** with your claims (the same as Option 1). Then, create a **custom view** (inheriting from `TokenObtainPairView`), setting its `serializer_class` attribute to your custom serializer. Finally, update your **`urls.py`** to use this custom view instead of the default `TokenObtainPairView`.
- **Result:** You override the behavior at the view level, gaining complete control over the token generation and the HTTP response.
- **Key Detail:** This approach is often preferred when you need **fine-grained control**, such as ensuring display claims (like `username`) are **only** added to the short-lived **Access Token**, not the sensitive, long-lived Refresh Token.

### Implementing option 1

1. Create a `simple_jwt.py` file in your `users` app folder.

Then import the `TokenObtainPairSerializer` serializer:

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
```

In documentation, it has also imported `TokenObtainPairView` like this:

```python
from rest_framework_simplejwt.views import TokenObtainPairView
```

But we don't need it (it's from the older documentation and we will discuss it in option 2).

2.  **Create a Custom Serializer:** Copy and past this code in your `simple_jwt.py` file

```python
class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims
        token['username'] = user.username
        token['message'] = 'Hello world'
        # ...

        return token
```

Here we've inherited from `TokenObtainPairSerializer` and overridden the `get_token` method, which is responsible for creating the initial token payload.

We access the authenticated `user` object passed to the method to retrieve data (like `user.username`).

3. Configure Simple JWT:
   Navigate to settings.py file in project package and override `TOKEN_OBTAIN_SERIALIZER` like this:

```python
SIMPLE_JWT = {
  # It will work instead of the default serializer(TokenObtainPairSerializer).
  "TOKEN_OBTAIN_SERIALIZER": "users.simple_jwt.MyTokenObtainPairSerializer",
  # ...
}
```

We are overriding the SimpleJWT's default `TOKEN_OBTAIN_SERIALIZER` with our custom serializer.

Note: Previously we had this SimpleJWT configuration, if you remember:

```python
SIMPLE_JWT = {
  "ACCESS_TOKEN_LIFETIME": timedelta(days=5),
    }
```

Now add what I mentioned above.

Note: From Simple JWT's documentation:

Note that the example above will cause the customized claims to be present in both refresh and access tokens which are generated by the view. This follows from the fact that the get_token method above produces the refresh token for the view, which is in turn used to generate the view’s access token.

**Testing the Custom Claims**

After generating a new token, when you paste the **access token** into [jwt.io](http://jwt.io), the decoded payload now includes your custom fields: `username` and `message`. This demonstrates how to encode any user information directly into the token.

---

### Implementing option 2

Note: `Customizing token claims` documentation I see now is different from what tutor displays in the video.

In the video, the documentation (and ofcourse the tutor) creates a custom view too. So create a `simple_jwt.py` file in project package or any other prefered folder and past this code inside it:

```python
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer
from rest_framework_simplejwt.views import TokenObtainPairView

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)

        # Add custom claims
        token['username'] = user.username
        token['message'] = 'Hello world'
        # ...

        return token

class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

In this implementation, the old documentation has created a custom view which inherits from `TokenObtainPairView` and only overrides `serializer_class` with our new `MyTokenObtainPairSerializer`.

Then we'll use this view in our urls.py file instead of the default `TokenObtainPairView`. Is it clear?

```python
from .simple_jwt import MyTokenObtainPairView

urlpatterns = [
    # The following two lines are for Simple JWT library
    path('api/users/login/', MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
    # path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

Note: In new documentation, the custom view has been removed:

```python
class MyTokenObtainPairView(TokenObtainPairView):
    serializer_class = MyTokenObtainPairSerializer
```

But the related import is still there!!!:

```python
from rest_framework_simplejwt.views import TokenObtainPairView
```

I just wanted to mention it so you don't get confused.

We've imported `TokenObtainPairView` here and we'll use it here, so remove this import from urls.py file. We don't need it there anymore.

**Testing the Custom Claims**

After generating a new token, when you paste the **access token** into [jwt.io](http://jwt.io), the decoded payload now includes your custom fields: `username` and `message`. This demonstrates how to encode any user information directly into the token.

---

## 2\. Customizing the API Response Data

The default login response only returns `access` and `refresh` tokens as a JSON object like this:

```json
{
    "access": ... ,
    "refresh": ...
}
```

. To simplify frontend state management, we can modify the response to immediately return core user data. This avoids the need for an extra API call or client-side token decoding.

### Steps to Customize the Response

Note: I couldn't find the next stuff in Simple JWT documentation which is very weird. The tutor is showing code on a Github account named SimpleJWT but I couldn't find the repo he was seeing.

1.  **Modify the Custom Serializer:** Go back to `MyTokenObtainPairSerializer` and override the **`validate`** method. This method is called _after_ authentication and _before_ returning the final response.

```python
class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    # ... (get_token method from above is still here) ...

    def validate(self, attrs):
        # Call the original validate method to get the tokens
        data = super().validate(attrs)

        # Retrieve the user object, which is available in self.user
        user = self.user

        # Add custom user data to the response dictionary
        data['username'] = user.username
        data['email'] = user.email

        # The original tokens (access and refresh) are already in 'data'
        return data
```

2.  **Testing the Response:** When you hit the `/api/users/login/` endpoint with a `POST` request, the new response will look like this:

    ```json
    {
      "refresh": "...",
      "access": "...",
      "username": "dennis",
      "email": "dennis@email.com"
    }
    ```

By overriding the `validate` method, we successfully inject additional user information directly into the login response, making it easier for the frontend to update its application state immediately.

In the next video, we will look at how to properly serialize and return a more complete **user profile** object in this response.

## Video: User Serializer

This guide covers creating a API endpoint to fetch the user's profile and customizing the login response to include full user data and a new token.

#### 1\. Create the User Profile Serializer

We need a dedicated serializer to control exactly which user fields are exposed via the API.

Create a `serializers.py` in `users` app directory and past the following code inside it:

```python
from rest_framework import serializers
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        # Fields to expose:
        fields = ['id', 'username', 'email']
```

#### 2\. Create the User Profile View

This view will use the `request.user` object (populated by the JWT) to return the authenticated user's data.

Create a `views.py` file in your `users` app folder and past this code in it:

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
# Assuming UserSerializer is imported from your serializers file
from .serializers import UserSerializer

# Define the view for the profile endpoint
@api_view(['GET'])
def getUserProfile(request):
    user = request.user

    # Serialize the single user object
    serializer = UserSerializer(user, many=False)

    return Response(serializer.data)
```

`Question`: Where this `user` object on request (request.user) came from? We didn't have this in `getProducts` view for example.

`Answer`: The request.user object exists in both your getProducts and getUserProfile views because the Django REST Framework (DRF) authentication process runs on every request, regardless of the view.

You simply don't notice the request.user in getProducts because you never use it. Your code works perfectly whether the user is logged in or anonymous:

```python
# getProducts: Doesn't need the user
def getProducts(request):
    # This line runs fine for both AnonymousUser and real User
    products = Product.objects.all()
    # ...
```

In contrast, your getUserProfile view is built around the assumption that the user is authenticated. It immediately relies on the successful outcome of the JWT authentication process to access the required User object.

```python
# getUserProfile: Relies on the user
def getUserProfile(request):
    # This line requires a real User object to succeed cleanly
    user = request.user
    # ...
```

In summary, the request object always has a .user attribute (either an AnonymousUser or a real User instance). You only care about it, and see its effects, in views that deal with private user data or require authentication.

`More explanation`: Whenever a request is sent to a Django REST Framework (DRF) API endpoint, a sophisticated process takes place behind the scenes to verify the user's identity:

Token Check: DRF and Simple JWT combined authentication logic automatically checks the request's Authorization header for a bearer Access Token.

Token Validation: The Simple JWT library validates the token's signature, ensures it hasn't expired, and decodes its payload to retrieve the user's unique ID.

User Retrieval & Attachment: Using this ID, the system fetches the corresponding User model object from the database. This complete User object is then attached directly to the incoming request object.

The Result:

Due to this automated process, by the time your view function starts executing, the authenticated user is readily available and accessible via request.user. This is why you can instantly use user = request.user in a protected view without manually handling token verification.

`Note`: We received both an 'access' token and a 'refresh' token after logging in. We actually received a JSON object like this:

```json
{
    "access": ...,
    "refresh": ...
}
```

Note that both values are JWTs, because both of them adhere to the JWT standard structure: they are base64-encoded strings separated by two periods (.). This structure contains a header, a payload (claims), and a signature.

The thing is, they serve different purposes:

The value associated with the `access` key (access token) is used for authentication and authorization in subsequent requests. This token is short-lived (e.g., 5-15 minutes) and is sent in the Authorization: Bearer <token> header of every protected API request (like your /api/users/profile/ view). It carries the user's identifier and is used to quickly verify the user's credentials and permissions.

The value associated with the `refresh` key (refresh token) is never sent to protected endpoints. Its sole job is to be sent to the refresh endpoint (e.g., /api/token/refresh/) to obtain a new access token after the current one has expired. It acts as a secure key to keep the user logged in without requiring them to re-enter their username and password.

#### 3\. Define the URL Path

Link the new view to a URL path. In `users` app folder, open url config file and add this path:

```python
from django.urls import path
from . import views

urlpatterns = [
    # Existing login/token path...
    # New path for user profile
    path('profile/', views.getUserProfile, name='user-profile'),
]
```

#### 4\. Testing the endpoint in browser

If we check `/api/users/profile` route in the browser, we get this:

```json
{
  "id": null,
  "username": ""
}
```

the reason is we are not sending a JWT with our GET request to this endpoint.

#### 5\. Testing the Endpoint using Postman

First navigate to `/api/users/login` in the browser and login to get a fresh `access` token. Copy the token (leave the `refresh` token, we don't need it now).

Then in Postman, we'll send a **GET** request to `/api/users/profile/` endpoint with an Authorization Header.

Go to the **Headers** tab and add:

    - **Key:** `Authorization`
    - **Value:** `Bearer <PASTE_YOUR_ACCESS_TOKEN_HERE>`

Note: The value for `Authorization` header is `Bearer` followed by a space and then past the access token.

Send the request. The response should now return the serialized user data (e.g., `id`, `username`, `email`,).

#### 6\. Update the user serializer

`Serializer methods` are custom functions defined within a Django REST Framework (DRF) serializer class that are used to compute a field's value dynamically, rather than pulling the value directly from the corresponding model instance. They are implemented using the SerializerMethodField and must follow the naming convention get_field_name.

We use them primarily to customize the output representation of our data. This allows us to combine multiple model fields (like combining first_name and last_name into a single full_name field), perform calculations (like an average_rating), or fetch data that doesn't exist on the main model being serialized (like an isAdmin status based on the user's is_staff attribute).

The method which implements a `SerializerMethodField` takes two mandatory arguments:

`self`: This refers to the serializer instance itself, allowing the method to access other fields, the serializer's context (like the request object), or its internal state.

`obj`: This is the object (model instance) being currently serialized. For example, in a UserSerializer, obj would be the specific User model instance being processed, allowing you to access its attributes like obj.username or obj.email.

So update the `UserSerializer` like this:

```python
from rest_framework import serializers
from django.contrib.auth.models import User

class UserSerializer(serializers.ModelSerializer):
    # On frontend, we used _id for all our data (users, products etc). We defined the Product model and other models to have an _id field as their primary key. But the User model is defined by Django. Changing it is annoying so what we can do instead is to use a serializer method to return an _id too.
    _id = serializers.SerializerMethodField(read_only=True)

    # Custom field to get the user's display name
    name = serializers.SerializerMethodField(read_only=True)

    # In Django User model, we have a is_staff field indicating the a user is admin. But in frontend, we used isAdmin field instead, so I want to return such a field too.
    isAdmin = serializers.SerializerMethodField(read_only=True)

    class Meta:
        model = User
        # Fields to expose: ID (aliased), username, email, display name, and admin status
        fields = ['id', '_id', 'username', 'email', 'name', 'isAdmin']

    # --- Serializer Methods ---
    #* In DRF, we must start serializer method names with `get` followed by an `_` and then the name of field we want to return. This is mandatory.

    # 1. Get the primary key as '_id' for frontend consistency
    # Because _id has and `_` itself, we have two underscores here in get__id
    def get__id(self, obj):
        # obj here is the User instance
        return obj.id

    # 2. Get a single display name (using first_name if available, otherwise email)
    def get_name(self, obj):
        name = obj.first_name
        # If there's no first_name for the user, return their email (we're sure they have an email because the email is required)
        if name == '':
            name = obj.email
        return name

    # # 3. Simplify the 'is_staff' check for the frontend
    def get_isAdmin(self, obj):
        return obj.is_staff
```

#### 7\. Create user serializer with token

Next, we'll create another user serializer. This new serializer will function identically to our current one, but it will also generate a fresh token. We need this for scenarios like initial user registration or when a user resets their account information, as a new token is required to represent the updated user state.

Copy and past this serializer after current one in `serializers.py` file in `users` app folder:

```python
from rest_framework_simplejwt.tokens import RefreshToken

class UserSerializerWithToken(UserSerializer):
    token = serializers.SerializerMethodField(read_only=True)

    class Meta:
        model = User
        fields = ['id', '_id', 'username', 'email', 'name', 'isAdmin', 'token']

    def get_token(self, obj):
        token = RefreshToken.for_user(obj)
        return str(token.access_token)
```

As you see the `UserSerializerWithToken` is a subclass of `UserSerializer`, so here we'll have all the attributes of `UserSerializer` plus one, which is the `token`.

Also note that in this serializer method,

```python
def get_token(self, obj):
    token = RefreshToken.for_user(obj)
    return str(token.access_token)
```

this `RefreshToken.for_user(obj)` returns an object named `token` which includes both an access token and a refresh token and we're just returing the string value of the access token, right?

the `UserSerializerWithToken` returns a dictionary and the value for the key `token` is an `access token`, like the one we get when logging in using `/api/users/login`. Is it clear?

Next open `simple_JWT.py` file and update the `validate` method in `MyTokenObtainPairSerializer` like this:

```python
# Other imports ...
from .serializers import UserSerializerWithToken

class MyTokenObtainPairSerializer(TokenObtainPairSerializer):
    # @classmethod
    # def get_token(cls, user):
    # ...

    def validate(self, attrs):
        data = super().validate(attrs)

        serializer = UserSerializerWithToken(self.user)
        for k, v in serializer.data.items():
            data[k] = v

        return data
```

Previously when we logged in using `/api/users/login` endpoint and our credentials, we would get an access token and a refresh token. Our API endpoint JSON response was like this:

```json
{
    "access": ...,
    "refresh": ...
}
```

Now we've changed the above serializer and we respond with other fields including a `token` field which its value is an `access token` (I explained the reason earlier). So we'll end up with 3 JWTs in our JSON response:

```json
{
  "access": ...,
  "refresh": ...,
  "token": ...,
  // Other fields ...
}
```

We can use any of values of `access` and `token` fields, because both of them are access tokens. Clear?
But I still don't know why we returned another access token, we had one before!

#### Review: Why we need serializers

We need a serializer for `product` object in `product = Product.objects.get(_id=pk)` (as an example) because the `product` is a complex Django model instance that cannot be sent directly across the internet.

When you execute `product = Product.objects.get(_id=pk)`, you retrieve a feature-rich Python object that contains methods, database connection info, and relational data.

The primary reason we need a serializer is to transform this complex Python object into a simple, standardized data format (like a Python dictionary or list) that can then be easily and universally rendered as JSON for the API response. Without serialization, the web browser or client application would not be able to understand or safely interpret the data structure of the Django model instance.

Using a serializer in Django REST Framework (DRF) is very similar to using the .lean() method in Mongoose for MongoDB:

Stripping the "Complexity":

Mongoose (.lean()): When you call Product.find().lean(), Mongoose returns simple, plain JavaScript objects (POJOs) instead of full Mongoose Documents. It strips away all the Mongoose-specific overhead, such as change tracking, validation, virtuals, and methods.

DRF Serializer: A serializer takes a Django Model instance and strips away all the complex Python object features (methods, database session information, relationships) to produce a simple Python dictionary that is ready to be converted into JSON.

Focus on Data:

In both cases, the goal is to get the raw data needed for the API or view with the least amount of overhead, resulting in better performance and a clean, predictable output.

We normally use a serializer in a view to format complex data typically a Django model instance (or a queryset of instances), and translate it into native Python data types (like dictionaries, lists, strings) so it can be easily rendered into JSON for an API response.

However, here we are using the UserSerializerWithToken inside the MyTokenObtainPairSerializer's validation method. While it may sound confusing, this is a valid and powerful DRF pattern: we're essentially using one serializer as a helper to package customized user data and a new token, which is then cleanly merged into the final authentication response generated by the main token serializer.

Note: The result of `products = Product.objects.all()` is a QuerySet, not a standard Python list. Here's what that means and the crucial difference:

#### What is a QuerySet?

A QuerySet is a special object provided by Django's Object-Relational Mapper (ORM). It represents a database query but does not execute that query immediately.

Lazy Evaluation: It acts as a promise or a blueprint for the data you want. The actual database query is only run when you iterate over the QuerySet or explicitly request its data (e.g., calling len(), converting it to a list, or slicing it).

Chainability: Because the query hasn't run, you can chain additional filters and ordering methods onto the QuerySet:

```Python
# A new QuerySet is returned, still no database hit
active_products = products.filter(is_active=True).order_by('name')
```

A serializer itself doesn't convert a QuerySet into a list; rather, it converts the model instances within the QuerySet into a list of Python dictionaries.

Here's the distinction:

The Serializer's Role: The serializer's job is data transformation, not data structure conversion. It takes the model instances and runs the serialization logic to convert each one into a dictionary.

The many=True Role: When you pass a QuerySet to a serializer, you must use the many=True argument:

```Python
# products is a QuerySet
serializer = ProductSerializer(products, many=True)

# serializer.data is a list of dictionaries
final_data = serializer.data
```

The many=True argument tells the serializer: `I am giving you a collection (like a QuerySet or list), so you need to iterate over every item and apply the serialization rules to each one.`

The final output (serializer.data) will be a list of dictionaries, with each dictionary representing a single, serialized product. The conversion from the model instances to a list of dictionaries is what makes the data ready to be rendered as JSON.

#### Note: Django ORM: QuerySet vs. Single Model Instance

##### The Question

What is the difference in the object type returned by `Product.objects.all()` and `Product.objects.get(pk=1)`?

##### The Answer

The results of these two methods are fundamentally different:

| Method                      | Result Type        | Multiplicity | Database Hit | Analogy                                                                                  |
| :-------------------------- | :----------------- | :----------- | :----------- | :--------------------------------------------------------------------------------------- |
| `Product.objects.all()`     | **QuerySet**       | Zero or Many | **Lazy**     | A **blueprint** or **database promise** for a list of objects; the query hasn't run yet. |
| `Product.objects.get(pk=1)` | **Model Instance** | Exactly One  | **Eager**    | A single, fully loaded **Python object** representing a specific database record.        |

In summary, `Product.objects.all()` gives you a **QuerySet** (the definition of a query), whereas `Product.objects.get(pk=1)` gives you a single, concrete **Product model instance** (the result of an immediate database retrieval).

## Video: Protected Routes

Now, we want to set some restrictions on our views. Our `getUserProfile` view here that gets our user profile, it does actually restrict you if you don't have a proper token, but the view itself is not actually restricted to anything:

```python
@api_view(['GET'])
def getUserProfile(request):
    user = request.user
    serializer = UserSerializer(user, many=False)
    return Response(serializer.data)
```

It fails if you don't have a token. But technically anybody can access this view. Pay attention to the difference that now it's protected in a way because it fails if you don't have proper token, so any one can access this view but fails without a token. But we want it to be inaccessible if there's no proper token with the request.

There's certain views later on down in the future that are going to need to be protected. So what we're going to do is add in another decorator over here and then set the permission level.

we're going to have two permission levels. The first is going to be for a user to be authenticated. So a user must be authenticated before accessing a certain view. And then another permission level is going to be the fact that a user must be an admin.

Setting permission classes for views in Django Rest Framework (DRF) is crucial for securing your API endpoints. This tutorial shows how to use the `permission_classes` decorator and different permission levels (`IsAuthenticated`, `IsAdminUser`) with SimpleJWT.

#### 1\. Import Necessary Modules

Before setting up any restrictions, you need to import the required decorators and permission classes from DRF.

Open views.py in `users` app folder and import the following stuff:

1.  **Import `permission_classes` decorator:** Import this decorator, typically alongside `api_view`.
2.  **Import Permission Classes:** Import the specific permission classes you need.

```python
from rest_framework.decorators import api_view, permission_classes # Import permission_classes
from rest_framework.permissions import IsAuthenticated, IsAdminUser # Import the permission levels
```

`IsAuthenticated`: this means a user must be authenticated (logged in) to access this view.

`IsAdminUser`: this means a user must be an admin to access this view; in other words, the field `is_staff` must be True in the user database table.

#### 2\. Restrict a View to Authenticated Users

The `IsAuthenticated` permission class ensures that a user must provide a valid token (i.e., be logged in) to access the view.

#### Securing the `getUserProfile` View

Apply the `@permission_classes` decorator directly above the function-based view, passing the required permission(s) in a list.

1.  **Apply the decorator** to the `getUserProfile` view.
2.  **Set the permission** to `IsAuthenticated`.

```python
@api_view(['GET'])
@permission_classes([IsAuthenticated]) # <--- Add this decorator
def getUserProfile(request):
    # ... view logic for getting the user profile ...
    pass
```

#### Verification (Using Postman):

- **Without a token:** The request should fail with an error like "Authentication credentials were not provided.":

```json
{
  "detail": "Authentication credentials were not provided."
}
```

- **With a valid token:** The request should succeed, returning the user's profile data like this:

```json
{
  "id": 1,
  "_id": 1,
  "username": "fardin",
  "email": "fardin@email.com",
  "name": "Fardin",
  "isAdmin": true
}
```

---

#### 3\. Restrict a View to Admin Users Only

The `IsAdminUser` permission class ensures that only users who are both **authenticated** and have their **`is_staff` status set to `True`** can access the view.

#### Securing the `getUsers` View

An endpoint to retrieve a list of all users should typically be restricted to administrators.

1.  **Create the `getUsers` view** (which requires importing the `User` model and a `UserSerializer`).
2.  **Apply the decorator** to the new view.
3.  **Set the permission** to `IsAdminUser`.

<!-- end list -->

Add this view in `views.py` in `users` app folder:

```python
from django.contrib.auth.models import User

@api_view(['GET'])
@permission_classes([IsAdminUser]) # <--- Restricts access to only admin (is_staff=True) users
def getUsers(request):
    users = User.objects.all()
    serializer = UserSerializer(users, many=True)
    return Response(serializer.data)
```

**Note:** You **don't** need to include `IsAuthenticated` alongside `IsAdminUser`, as being an admin **implies** being authenticated.

Then add a path in `urls.py` file `users` app like this:

```python
urlpatterns = [
    path('', getUsers, name='users'), # This is for newly created getUsers view
    path('profile/', getUserProfile, name='user-profile'),
    # The following two lines are for Simple JWT library
    path('login/',MyTokenObtainPairView.as_view(), name='token_obtain_pair'),
    # path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

#### Verification (Using Postman):

- **Non-Admin User Token:** The request should fail with a permission error.
- **Admin User Token:** The request should succeed, returning the list of all users.

## Video: Register User

This section outlines how to create a **User Registration API endpoint** in Django Rest Framework (DRF) that accepts a POST request, creates a new user, hashes the password securely, and returns the new user data along with their authentication token. It also includes adding a custom error handler for duplicate email registration.

#### 1\. Prepare Imports and View Structure

You need to import the necessary tools for API views, responses, password hashing, and handling HTTP status codes.

1.  **Import the `make_password` function** from Django's authentication utilities.
2.  **Import `status`** from `rest_framework` for custom error responses.
3.  **Define the initial `registerUser` view** as a function-based view that only accepts POST requests.

<!-- end list -->

Copy this new view in `views.py` file in `users` app:

```python
from rest_framework import status # For custom error messages
from django.contrib.auth.hashers import make_password # To hash passwords securely
from .serializers import UserSerializerWithToken # Make sure your custom serializer is imported

# ... (Other imports like User and Serializers) ...

@api_view(['POST']) # Restrict this view to only POST requests
def registerUser(request):
    data = request.data # Get data from the request body
    try:
        user = User.objects.create(
            first_name=data['name'],
            username=data['email'],
            email=data['email'],
            password=make_password(data['password'])
        )

        serializer = UserSerializerWithToken(user, many=False)
        return Response(serializer.data)
    except:
        message = {'detail': 'User with this email already exists'}
        return Response(message, status=status.HTTP_400_BAD_REQUEST)
```

#### Note: we used `UserSerializerWithToken` here, not just `UserSerializer`, because later we need the token to login the user just after the registration.

#### Implementing User Creation Logic

The core logic involves extracting data from the request body, hashing the password, and attempting to create the user object. We'll use a `try...except` block for error handling.

1.  **Wrap the creation logic in a `try...except` block** to catch database errors (like unique constraint violation for the email/username).
2.  **Use `User.objects.create()`** to save the new user.
3.  **Hash the password** using `make_password()` before saving it.
4.  **Use the email as the username** to simplify the registration process.
5.  **Serialize and return** the newly created user along with their token using our custom serializer (e.g., `UserSerializerWithToken`).

<!-- end list -->

#### Error handling

The User.objects.create(...) call requires the username and email fields to be unique. If a database integrity error occurs (e.g., trying to save a duplicate email), the application will crash with an unhandled exception (a django.db.utils.IntegrityError).

The current except: block is designed to catch this raw error and replace it with a meaningful API response.

We used `status` from DRF. Using bare status codes in your responses isn't recommended. REST framework includes a set of named constants that you can use to make your code more obvious and readable.

`status.HTTP_400_BAD_REQUEST` evaluates to the number `400`. If you simply print the `status.HTTP_400_BAD_REQUEST`, you get the number `400`.

Using the constant does not change the numeric status code sent in the HTTP response header; both 400 and status.HTTP_400_BAD_REQUEST result in the same code being sent.

The essential reason to use the DRF constant is for code quality and maintainability:

1. Readability: The constant is self-documenting, instantly clarifying the intent of the status code (e.g., that the server is reporting a "Bad Request").

2. Safety & Consistency: It prevents subtle bugs. If a developer accidentally types an incorrect number (e.g., 40 instead of 400), the code will likely crash immediately with a clear NameError instead of deploying with a hard-to-debug, invalid status code.

3. Convention: It adheres to established best practices within the Django and DRF communities

You can read more about it in documentation using this [link](https://www.django-rest-framework.org/api-guide/status-codes/)

#### 3\. Configure the URL Path

The registration view needs a dedicated URL endpoint, typically separated from general user endpoints.

1.  **Open your `urls.py`** in `users` app.
2.  **Define a new path** for the `registerUser` view.

<!-- end list -->

```python
from .views import registerUser

urlpatterns = [
    # ... (Other paths like 'users/profile/' and 'users/') ...

    # New registration path
    path('register/', registerUser, name='register'),
]
```

---

#### 4\. Test the Registration Endpoint (Postman)

Test the new endpoint to ensure user creation and error handling work correctly.

1.  **Set the Request Method:** POST.
2.  **Set the Request URL:** `http://127.0.0.1:8000/api/users/register/`.
3.  **Configure the Request Body:**
    - Go to the **Body** tab.
    - Select **form-data**.
    - Add the following key-value pairs:
      | Key | Value |
      | :--- | :--- |
      | `name` | John Doe |
      | `email` | john.doe@example.com |
      | `password` | secure123 |
4.  **Send the Request:**
    _ **On Success (New User):** You should receive a response with the new user's details and an `access` (or `token`) value.
    _ **On Failure (Duplicate Email):** If you send the same email again, you should receive the custom error message: `{"detail": "User with this email already exists."}` with an **HTTP 400 Bad Request** status code.

Note: When we make a POST request to `http://127.0.0.1:8000/api/users/register` without a `/` at the end (a trailing slash), we get an error and after adding the trailing slash, the error is gone and we get the response from the API endpoint. This is the error:

`You called this URL via POST, but the URL doesn't end in a slash and you have APPEND_SLASH set. Django can't redirect to the slash URL while maintaining POST data. Change your form to point to 127.0.0.1:8000/api/users/register/ (note the trailing slash), or set APPEND_SLASH=False in your Django settings.`

Here is the explanation:

That error message is telling you that **Django is trying to be helpful but can't be because of the HTTP method (POST)**.

Here is what the message means in simple terms:

| Term/Setting                      | Simple Explanation                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| :-------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Trailing Slash (`/`)**          | The final forward slash at the end of a URL (e.g., `/register/`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **`APPEND_SLASH=True` (Default)** | This is a setting in your Django project. By default, it's set to `True`, which means **Django expects all URLs to end with a slash (`/`)**. If you hit a URL _without_ a slash, Django automatically tries to **redirect** you to the same URL _with_ the slash.                                                                                                                                                                                                                                                                                                               |
| **The Problem (Why POST Fails)**  | When you use a **GET** request (to view a page), a redirect works fine. However, when you use a **POST** request (to _send data_ to create or update something): 1. Django receives your POST data (e.g., name, email, password). 2. It decides the URL needs a slash (`/`). 3. It tries to **redirect** your request. 4.**A redirect turns the request into a GET request.** This is a security and design feature of HTTP. 5. **Result:** Your POST data (the name, email, password) is lost during the redirect, and the registration view never receives the data it needs. |
| **The Solution**                  | The error suggests one of two things: 1. **The best fix:** Change your Postman request URL to include the trailing slash: `http://127.0.0.1:8000/api/users/register/`. 2. **Alternative fix:** If you absolutely don't want trailing slashes, you can turn off the automatic redirect feature by setting `APPEND_SLASH = False` in your project's `settings.py` file. (Most developers just stick to the convention and use the trailing slash).                                                                                                                                |

In short: **Your Django project is configured to require a trailing slash. Because you sent a POST request without one, Django couldn't automatically fix the URL for you without losing your data, so it failed with that specific error.**

---

### Let's look at request.data

In our view, we used `request.data`:

```python
#...

@api_view(['POST']) # Restrict this view to only POST requests
def registerUser(request):
    data = request.data # Get data from the request body
    # ...
```

Let's print what's inside it:

```
<QueryDict: {'name': ['Payam Pak'], 'email': ['payam@email.com'], 'password': ['fardin72']}>
```

#### So why Django Request Data is in Lists (`QueryDict`)

#### Question

When printing `request.data` in a Django Rest Framework (DRF) view (e.g., for user registration), the output looks like a dictionary but the values are enclosed in lists, such as: `<QueryDict: {'name': ['Payam Pak'], 'email': ['payam@email.com']}>`. Why are the values in a list instead of just a single string?

#### Answer

The values are stored in lists because the object used by Django to handle incoming HTTP data, called the **`QueryDict`**, is designed to be **multi-valued** by default.

1.  **Multi-Value Field Support:** This structure is necessary to correctly handle standard web form submissions where a single input name (key) can send multiple values. A common example is a group of checkboxes where a user can select several options simultaneously (e.g., `color=red&color=blue`).
2.  **Preventing Data Loss:** By storing all associated values in a list (e.g., `['red', 'blue']`), Django ensures that no data is silently lost, even if only one value is expected.
3.  **Accessing the Value:** In common API scenarios (like JSON payloads), the list usually contains only one element. You can safely access the value by indexing the list (`request.data['key'][0]`), although Django Rest Framework (DRF) provides conveniences that allow you to use `request.data['key']` directly to retrieve the single string, like what we've done in the view. In other words, DRF has provided the convenice that `request.data['key']` would be equal to `request.data['key'][0]`.

---

### A note about VSCode:

The bottom bar in VS Code is called the **Status Bar**.

The section that displays information like the Python version and environment name, and which opens the interpreter list when clicked, is called the **Language Status Item** (or specifically, the **Python Interpreter Status Item**).

#### 1. Status Bar

The **Status Bar** is the thin blue bar located at the very bottom of the VS Code window.

It displays information relevant to the currently active file and workspace, including:

- **Line and Column Number** (e.g., `Ln 39, Col 20`)
- **Indentation Size** (e.g., `Spaces: 4`)
- **Encoding** (e.g., `UTF-8`)
- **End-of-Line Sequence** (e.g., `CRLF`)
- **Language Mode** (e.g., `Python`)

#### 2. Python Interpreter Status Item

This specific element is a crucial piece of the **Status Bar** for Python development.

- **Name:** It's generally referred to as the **Python Interpreter Selector** or the **Python Environment Indicator**.
- **Function:** It shows which Python environment/interpreter VS Code is currently using for the workspace. In your image, it shows **`3.12.5 (.venv)`**, indicating that the Python executable inside your local virtual environment is active.
- **Action:** Clicking this item triggers the **"Select Interpreter"** command, allowing you to easily switch between your various Python installations (like your `.venv`, Conda environments, or Global Python).

---

### Setting Up Ruff for Python and Django in VS Code

For maintaining code quality and style in Python and Django projects, the general choice is between **Black** (formatting only) and **Ruff** (comprehensive linting and formatting). **We recommend Ruff.**

#### 1\. Understanding the Tools

- **Black:** Primarily a **code formatter**. It strictly enforces style (like spacing and line breaks) but provides **no linting** (it doesn't check for errors or quality issues).
- **Ruff:** An **all-in-one tool** that provides both **fast linting** (error checking, style warnings) and **formatting**.

#### 2\. Installation and Setup

Install the official **Ruff extension** (Publisher: Charlie Marsh) from the VS Code Marketplace. By installing Ruff or Black Formatter extensions, the corresponding libraries ship with them too. Don't install the libraries yourself too.

#### 3\. Configure VS Code Settings

To keep your global Prettier settings for frontend files while enabling Ruff for Python, use a language-specific override in your `settings.json`:

```json
{
  // Global setting: Prettier is the default formatter for all file types (JS, JSON, etc.)
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  // LANGUAGE OVERRIDE: This block overrides the defaultFormatter rule ONLY for Python files
  "[python]": {
    "editor.defaultFormatter": "charliermarsh.ruff"
  }

  // ... other settings ...
}
```

#### 4\. Troubleshooting Formatting (If it Fails)

If the code doesn't format after following the steps above, the issue is almost always that VS Code is using the wrong Python environment. The recommended Python interpreter is the one in your virtual environment folder of your current project ofcourse, and VSCode automatically selects this one, but if this one doesn't work and another one does, you have done something wrong like installing Ruff library manually too (I mentioned don't install it)

- **Check the Status Bar:** Look at the **Status Bar** (bottom bar) of VS Code. The **Python Interpreter Selector** (e.g., `3.12.5 (.venv)`) shows which environment is active.
- **Fix the Environment Mismatch:** Click on the interpreter selector and manually select the environment where you installed Ruff (e.g., the local project's `.venv`). Once the correct environment is selected, the formatter will find the Ruff executable and begin working.

---

## Video: Login with Email

### Configuring Email Login with Django Signals

Now, it's time to **configure our user model to allow login using an email address instead of a username**.

There are different ways to approach this. We could **override the default Django user model** and explicitly tell it to use the email field. However, to avoid fully customizing the model and overriding database schema, I'm going for a simpler solution. This is helpful if you still want to preserve the option of using a username.

The goal is to ensure the **email address is always the same as the username**.

When a user, like Dennis, was created manually, we have a distinct username and email, meaning we can't log in with the email. However, when we create a new user, like John, via the registration view, we're already saving the email value as the username. This allows for email-based login because the email is now stored in the `username` field.

To make sure this synchronization always happens, even if we update the email field later, we're going to use **Django Signals**.

You can read about Django signals [here](https://docs.djangoproject.com/en/5.2/topics/signals/) in documentation.

---

### Understanding Django Signals

Django signals consist of **senders** and **receivers**. Essentially, they allow our program to **listen for actions**.

In this scenario, whenever the user model is saved (when we hit **save**), a signal is **sent out**. This signal is received by a **receiver function**, which can then trigger a subsequent action on the backend. It listens for an update in the database and then triggers another action.

We'll be working with two main types of signals: **pre-save** and **post-save**.

I'll start with **`pre_save`**. This signal fires off _before_ the model is actually saved, alerting us that the model is about to be saved. We'll create a function that runs at this moment to update the `username` field. Alternatively, the **`post_save`** signal fires off _after_ the item has been successfully updated.

That's the quick overview. I strongly recommend checking out the 13-minute video on my channel for a more in-depth explanation and a solid example. Now, let's start building it out. I'll still explain the steps as we go, but those resources will provide a richer context.

---

### Implementing the Signal

In our `users` app's directory, let's create a new file called `signals.py`.

#### 1\. Imports

In `signals.py`, we need two main imports:

1.  The `pre_save` signal:
    ```python
    from django.db.models.signals import pre_save
    ```
    This allows us to listen for the model-save process and fire an action _before_ the save finishes.
2.  Our `User` model:
    ```python
    from django.contrib.auth.models import User
    ```
    We want to trigger an action whenever this specific model is created or updated.

#### 2\. The Receiver Function

Next, we define the function that will fire off when the user model is saved:

```python
def update_user(sender, instance, **kwargs):
    # sender is the model class (User in this case)
    # instance is the actual object being saved
    # kwargs are other keyword arguments
    # ...
```

For now, let's add a test print statement:

```python
from django.db.models.signals import pre_save #    This allows us to listen for the model-save process and fire an action before the save finishes.
from django.contrib.auth.models import User # We want to trigger an action whenever this specific model is created or updated.

# sender is the model class (User in this case)
# instance is the actual object being saved
# kwargs are other keyword arguments
# ...
def update_user(sender, instance, **kwargs):

    print("Signal triggered")
```

#### 3\. Connecting the Signal

To connect this function to the `pre_save` signal on the `User` model, we use the `connect` method:

```python
from django.db.models.signals import pre_save #    This allows us to listen for the model-save process and fire an action before the save finishes.
from django.contrib.auth.models import User # We want to trigger an action whenever this specific model is created or updated.

# sender is the model class (User in this case)
# instance is the actual object being saved
# kwargs are other keyword arguments
# ...
def update_user(sender, instance, **kwargs):

    print("Signal triggered")

pre_save.connect(update_user, sender=User)
```

This tells Django: when the `User` model is about to be saved, run the `update_user` function.

#### 4\. Configuring the App

We haven't connected our `signals` file to the app yet. We need to go into `apps.py` and add a `ready` method. So in `users` app, open `apps.py` and add this method:

```python
# ...

class UsersConfig(AppConfig):
    # ...
    def ready(self):
        # Import the signals file so Django knows about the connections
        import users.signals
```

For this to work correctly, make sure your app is installed in `settings.py` using its full configuration path:

```python
# In settings.py

INSTALLED_APPS = [
    # ...
    'users.apps.UsersConfig',
    # ...
]
```

Note: I didn't put the full configuration path in `settings.py` and still worked:

```python
# In settings.py

INSTALLED_APPS = [
    # ...
    'users',
    # ...
]
```

Note: the file you put your Django signals in does **not** have to be named `signals.py`. Django doesn't enforce this naming convention.

However, naming the file `signals.py` is an **extremely common and highly recommended convention** within the Django community for several reasons:

1.  **Readability and Discoverability:** Any developer looking at your app's directory knows immediately where the signal logic is defined.
2.  **Organization:** It keeps the signal handlers separate from models, views, and forms, making the application easier to maintain.

### Where to Put the Logic

Regardless of the file name, the crucial step is **making sure Django knows your signal handler exists**. This is typically done by importing the signal logic within your application's `AppConfig` file:

```python
# myapp/apps.py

from django.apps import AppConfig

class MyAppConfig(AppConfig):
    name = 'myapp'

    def ready(self):
        # Import your signal handlers here
        import myapp.signals # This is what tells Django to load the code
```

#### 5\. Testing the Connection

With everything saved, the signal should now fire off. When I go to the admin panel, edit Dennis, and hit **Save**, we can check the console.

There it is: **`Signal triggered`**. Our listener worked—it recognized the user model save and ran our function.

---

#### Finalizing the Fix

Inside our `update_user` function, we'll implement the "quick fix" for this login problem. While some may prefer fully overriding the user model, this approach is faster and has proven reliable in various applications.

When the signal is triggered (on save), we'll update the user's username using the email from the `instance`. So replace the previous testing `update_user` method with this:

```python
def update_user(sender, instance, **kwargs):
    # Check if the email field is not empty before proceeding
    user = instance
    if user.email:
        # Update the username with the email value
        user.username = user.email
```

With this in place, we no longer even need to provide a username during creation. The `pre_save` signal will fill the `username` field with the email _before_ the object is saved to the database.

Let's test this. My current username is "Dennis" and my email is "dennis2222@email.com". If I hit **Save and continue editing** after making a change, my **username is now `dennis2222@email.com`**.

If I change the email to `dennis@gmail.com` and save again, the username is automatically updated to `dennis@gmail.com`. This quick fix, utilizing the powerful capabilities of **Django Signals**, solves our login issue.

---

## Next Steps

### Note: About orders app

At the start of the course, we placed all models—such as Product, Order, OrderItem, ShippingAddress, and Review in the models.py file within the products app. We then created and applied the necessary database migrations.

I now want to create a separate app, named orders, to handle all order-related logic. However, moving the Order and OrderItem models from the products app to the new orders app would introduce significant difficulties with the existing database migrations and history.

Therefore, I have decided to keep the Order and OrderItem models in the products app and simply import them into the orders app whenever they are needed.

### Note: Understanding Django's Reverse Accessor (`_set`)

#### Question

In the `OrderSerializer` method `get_orderItems(self, obj)`, where `obj` is an `Order` model instance, what is `orderitem_set` in the line `items = obj.orderitem_set.all()`? I did not define a field with that name on the `Order` model.

#### Answer

`orderitem_set` is a special property called a **reverse accessor** that is **automatically generated by Django**.

1.  **Automatic Creation:** Django creates this property on the **target model** (`Order`) because the **source model** (`OrderItem`) has a `ForeignKey` pointing _back_ to the `Order` model (creating a one-to-many relationship).

2.  **Purpose:** The accessor allows you to easily retrieve a QuerySet of _all_ `OrderItem` objects associated with that specific `Order` instance (`obj`).

3.  **Naming Convention:** By default, Django names the reverse accessor using the pattern: **`source_model_name_lowercase`** + **`_set`**.

    $$\text{OrderItem (Model Name)} + \text{\_set} = \text{orderitem\_set}$$

If you want a different name (e.g., `items`), you must specify it using the **`related_name`** argument on the Foreign Key field in your `OrderItem` model:

```python
# In OrderItem model:
order = models.ForeignKey(Order, related_name='items', on_delete=models.CASCADE)
```

If you use related_name='items', your serializer method would look like this:

```Python
def get_orderItems(self, obj):
    items = obj.items.all() # Changed from obj.orderitem_set.all()
    serializer = OrderItemSerializer(items, many=True)
    return serializer.data
```

### Note: One-to-One Reverse Accessor

#### Question

In the `OrderSerializer`, why does the function `get_shippingAddress` use `obj.shippingaddress` when the `Order` model does not have a `shippingaddress` field?

```python
def get_shippingAddress(self, obj):
    address = ShippingAddressSerializer(obj.shippingaddress, many=False).data
    # ...
```

### Answer

The property **`obj.shippingaddress`** is the **One-to-One Reverse Accessor** automatically generated by Django.

1.  **Defining Relationship:** The field is defined on the **`ShippingAddress`** model, which has a `OneToOneField` pointing to **`Order`**:
    ```python
    order = models.OneToOneField(Order, on_delete=models.CASCADE, ...)
    ```
2.  **Reverse Access:** Because it's a `OneToOneField`, Django gives the target model (`Order`) a dynamic property to access the single, related instance from the source model (`ShippingAddress`).
3.  **Naming Convention:** For `OneToOneField`, the reverse accessor defaults to the **lowercase name of the source model**.
    $$\text{ShippingAddress (Model Name)} \rightarrow \text{shippingaddress}$$

This allows the `Order` instance (`obj`) to find the single associated `ShippingAddress` instance without needing an explicit field on the `Order` model itself.

<!-- Todo: Continue from here -->

We need to test these api endpoints:
POST /api/products/upload/
GET /api/orders/

Note: It's better to replace /api/products/upload/ sending product_id in request body with /api/products/:id/upload, sending product id as a path parameter.

## Merging React and Django for Deployment

This process involves integrating your React project into your Django project's folder structure, building the React project into static assets, and then configuring Django to serve these assets.

### Note:

When developing our backend in Django, we used `/api` before every API endpoint and we didn't use `http://localhost:8000/` for anything. That's because we want to use it to serve our frontend static files.

### 1\. File Structure Consolidation

The first step is to move the React project folder into the Django project's backend folder.

1.  **Stop Both Servers:** Turn off both your React development server (running on port 3000) and your Django server (running on port 8000). You can usually do this by pressing $\text{Ctrl}+\text{C}$ in the respective terminal windows.
2.  **Move the React Folder:** Take your React project folder (often named $\mathbf{frontend}$ or similar) and drag it into your main Django project folder (often named $\mathbf{backend}$ or similar, where $\text{manage.py}$ resides).
    - _Tip:_ If you have your text editor or command prompt open and **cd'd** into these directories, you might need to close them before the move is successful.

### 2\. Building React into Static Assets

Next, you'll use the React build command to transform your source code into production-ready static files.

1.  **Navigate to the React Directory:** Open your terminal, $\text{cd}$ into your Django project folder (e.g., $\text{cd backend}$), and then $\text{cd}$ into your React project folder (e.g., $\text{cd frontend}$).
2.  **Run the Build Command:** Execute the React build script.
    ```bash
    npm run build
    ```
    - This command will create a new folder named $\mathbf{build}$ inside your React project directory ($\text{frontend/build}$). This folder contains the minified HTML (an $\text{index.html}$ file), CSS, and JavaScript files—all the static assets that Django will now serve.

### 3\. Django Configuration: Templates

You need to tell Django where to find the main $\text{index.html}$ file generated by the React build process.

1.  **Open $\text{settings.py}$:** Open the $\text{settings.py}$ file of your Django project.

2.  **Import $\text{os}$:** Ensure you import the $\text{os}$ module at the top of the file if it's not already there.

    ```python
    import os
    ```

3.  **Configure $\text{TEMPLATES}$:** Modify the $\text{DIRS}$ list within the $\text{TEMPLATES}$ configuration to point to the $\mathbf{build}$ folder.

    - Find the $\text{TEMPLATES}$ section and update the $\text{DIRS}$ list like this (assuming your React folder is $\text{frontend}$):
      ```python
      TEMPLATES = [
          {
              # ...
              'DIRS': [os.path.join(BASE_DIR, 'frontend', 'build')],
              # ...
          },
      ]
      ```

### 4\. Django Configuration: URL Routing

Configure Django to use the $\text{index.html}$ as the homepage for the application.

1.  **Open $\text{urls.py}$:** Open the main $\text{urls.py}$ file of your Django project.

2.  **Import $\text{TemplateView}$:** Import the necessary class-based view.

    ```python
    from django.views.generic import TemplateView
    ```

3.  **Define the Homepage Route:** Add a path that points the root URL ($\text{path('')}$) to the $\text{index.html}$ file in your newly configured templates directory.

    - Replace or modify the default root path:
      ```python
      urlpatterns = [
          # ... other paths (like your API routes)
          path('', TemplateView.as_view(template_name='index.html')),
      ]
      ```
    - _Note:_ The $\text{template\_name}$ only needs to be $\text{'index.html'}$ because you already pointed Django to the containing $\text{build}$ folder in $\text{settings.py}$.

### 5\. Django Configuration: Static Files

Configure Django to locate the static CSS and JavaScript files that are referenced within the $\text{index.html}$ file.

1.  **Open $\text{settings.py}$:** Return to the $\text{settings.py}$ file.
2.  **Configure $\text{STATICFILES\_DIRS}$:** Add the $\text{STATICFILES\_DIRS}$ setting to point to the $\mathbf{static}$ folder within your $\mathbf{build}$ directory.
    ```python
    STATICFILES_DIRS = [
        # Othere static files directories like BASE_DIR / "static",
        os.path.join(BASE_DIR, 'frontend', 'build', 'static'),
    ]
    ```

### 6\. Verification and Deployment Readiness

Your Django project is now configured to serve the React application.

1.  **Start the Django Server:** In your terminal, navigate back to the main Django project directory (where $\text{manage.py}$ is) and start the server.
    ```bash
    python manage.py runserver
    ```
2.  **Check Port 8000:** Navigate to $\text{[http://127.0.0.1:8000/](http://127.0.0.1:8000/)}$ in your web browser. You should now see your React application.
3.  **Workflow for Changes:** If you want to make changes to your React code:
    - You can still run $\text{npm start}$ in the $\text{frontend}$ folder to develop with React's hot-reloading on **Port 3000**.
    - To see the changes reflected in the **Django server (Port 8000)**, you must **stop the Django server**, go back to the $\text{frontend}$ folder, run $\text{npm run build}$ again, and then restart the Django server.

## Switching from default SQLite database to PostgreSQL

Here is the breakdown of the process for migrating your Django project from SQLite to a production-ready PostgreSQL database, including setting up both a local connection and a live database on AWS RDS.

---

### 1\. Local PostgreSQL and pgAdmin Setup

Before configuring Django, you need the PostgreSQL server and a graphical management tool set up locally.

1.  **Download and Install:**
    - **PostgreSQL:** Download and install the PostgreSQL server for your operating system.
    - **$\text{pgAdmin}$ (GUI):** Download and install $\text{pgAdmin}$. This is a graphical user interface (GUI) that makes it easier to manage your PostgreSQL databases, tables, and data without using the command line.
2.  **Remember Credentials:** During the installation of $\text{pgAdmin}$, you'll set a master password and potentially a default username (often $\mathbf{postgres}$). **Write down your username and password**, as you will need them for both $\text{pgAdmin}$ and Django.
3.  **Create a Local Database in $\text{pgAdmin}$:**
    - Open $\text{pgAdmin}$.
    - Connect to the default server cluster (usually $\mathbf{PostgreSQL 1X}$ on $\mathbf{localhost:5432}$).
    - Right-click on **Databases** and select **Create** $\rightarrow$ **Database...**
    - Set the **Database** name (e.g., $\mathbf{proshop}$).

---

Of course\! You are absolutely right to ask about this. Before you can configure Django to use PostgreSQL, you need to install the necessary database adapter.

Here's the step to install `psycopg2`, which is essential for Django to communicate with your PostgreSQL database:

---

### 1.1\. Install the PostgreSQL Adapter for Python

Before making any changes to `settings.py`, you need to install the Python package that allows Django to interact with a PostgreSQL database.

1.  **Install $\text{psycopg2}$:** Open your terminal or command prompt (make sure your virtual environment is activated if you're using one) and run the following command:
    ```bash
    pip install psycopg2-binary
    ```
    - **Note:** `psycopg2-binary` is generally recommended for ease of installation, as it includes pre-compiled binaries. If you encounter issues or need specific features, `psycopg2` (which requires a C compiler and PostgreSQL development headers) is the alternative.
    - **Note:** The tutor installed `psycopg2` but Gemi ni told me to install `psycopg2-binary`.

---

### Note: When to use py -m before pip install package

In a virtual environment, the simple $\text{pip}$ command is sufficient.
When you are installing a Python library using $\text{pip}$, the choice between using the simple `pip install <package>` and the more explicit `py -m pip install <package>` (or `python -m pip install <package>` on Linux/macOS) depends almost entirely on whether you are working within an **active virtual environment**.

Here is the straightforward guide on when and why to use the explicit approach:

### 1. When You **Should Not** Use $\text{py -m}$

The explicit command is **unnecessary and discouraged** when your virtual environment (like one created with $\text{venv}$) is **already activated**.

- **The Command:** `pip install <package>`
- **Why:** When the environment is active, the system's path automatically points the simple $\text{pip}$ command to the specific $\text{pip}$ executable **inside that environment's folder**. Using $\text{py -m}$ can sometimes confuse the path or is simply redundant, adding unnecessary characters.

### 2. When You **Should** Use $\text{py -m}$

You should use the explicit command whenever you are **outside of a virtual environment** or when you need to be absolutely certain which Python interpreter's $\text{pip}$ you are executing.

- **Scenario 1: Installing Globally (Discouraged for Projects)**
  If you are installing a package system-wide (e.g., you are not in a virtual environment), using $\text{py -m pip}$ is the best practice to ensure you are running $\text{pip}$ as a module of the Python interpreter, rather than relying on a potentially broken or misconfigured system path link to the standalone $\text{pip}$ executable.
  - **The Command:** `py -m pip install <package>`
- **Scenario 2: Targeting a Specific Python Version**
  If you have multiple versions of Python installed (e.g., Python 3.9 and Python 3.11) and you want to install a package into a _specific_ global version without changing your system path.
  - **The Command:** `py -3.11 -m pip install <package>`
- **Scenario 3: Creating the Virtual Environment Itself**
  When you create the virtual environment, you run $\text{venv}$ as a module of a specific Python interpreter.
  - **The Command:** `py -m venv <name>`

### Summary for Your Django Development

Since you are consistently using **virtual environments** for your Django projects, you can rely on the simplest and cleanest command once your environment is active:

**`pip install psycopg2-binary`**

---

### 2\. Configure Django $\text{settings.py}$ for Local PostgreSQL

You need to change Django's database engine and provide the credentials to connect to the local PostgreSQL instance.

1.  **Open $\text{settings.py}$** in your Django project.

2.  **Comment Out/Replace SQLite Configuration:** Locate the $\text{DATABASES}$ setting and replace the default $\text{sqlite3}$ configuration with the PostgreSQL settings. **Note: This will disconnect your project from your old SQLite data.**

Replace this:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

With this:

```python
DATABASES = {
'default': {
    'ENGINE': 'django.db.backends.postgresql',# Change engine to postgresql
    'NAME': 'E-Commerce',# Your new database name (e.g., E-Commerce)
    'USER': 'postgres',# Your PostgreSQL username (e.g., postgres)
    'PASSWORD': 'fardin72',# Your PostgreSQL password
    'HOST': 'localhost',# Host for local connection
    'PORT': '5432',# Default PostgreSQL port
}
}
```

---

## 3\. Run Migrations on the New Local Database

Since you switched databases, you need to re-run migrations to create the Django tables in your empty PostgreSQL database.

1.  **Stop the Django Server:** If it's running, $\text{Ctrl}+\text{C}$ to stop the server.
2.  **Run Migrations:** Execute the $\text{migrate}$ command. (You don't need $\text{makemigrations}$ if your models haven't changed.)
    ```bash
    py manage.py migrate
    ```
    - **Verification:** You should see many tables (like $\text{auth\_user}$, $\text{base\_product}$, etc.) appear in $\text{pgAdmin}$ when you refresh the tables list under your $\text{E-Commerce}$ database schema.
3.  **Create a Superuser:** Create your admin user again, as the old user data was on the SQLite database.
    ```bash
    py manage.py createsuperuser
    ```
4.  **Restart Server and Test:** Run $\text{py manage.py runserver}$ and verify you can log in to the admin and your homepage loads (though it will be empty until you re-add products).

---

## 4\. Set up PostgreSQL on AWS RDS (Live Database)

For production, your database must be publicly accessible (not on your local machine). AWS Relational Database Service ($\text{RDS}$) is used for this.

1.  **Log in to AWS Console:** Sign in to your Amazon Web Services account.
2.  **Navigate to $\text{RDS}$:** Search for and go to the $\mathbf{RDS}$ dashboard (Relational Database Service).
3.  **Create a Database:**
    - Click **Create database**.
    - **Choose Standard Create** and select **PostgreSQL**.
    - **Select the Free tier** template for testing.
    - **Settings:**
      - **DB instance identifier:** Set a unique name (e.g., $\mathbf{proshop-identifier}$).
      - **Master username:** Set a custom username (e.g., $\mathbf{DennisIvy}$).
      - **Master password:** Set and confirm a strong password. **Save these credentials\!**
      - **DB instance class:** The Free tier automatically sets this to a $\mathbf{t2.micro}$ or $\mathbf{t3.micro}$ (Burstable).
      - **Public access:** Set this to **Yes**. This is crucial for connecting from outside AWS.
      - **Initial database name:** Set this to your desired database name (e.g., $\mathbf{proshop}$).
    - Click **Create database**. Wait approximately 5-10 minutes for the database status to change from **"Creating"** to **"Available"**.
4.  **Get the Endpoint:** Once available, click on your new database instance. The **Endpoint** is the public address (hostname) you'll use for the live connection. **Copy this value.**

---

## 5\. Configure Django for AWS RDS

Now you'll switch your Django project from the local PostgreSQL to the live AWS RDS instance.

1.  **Open $\text{settings.py}$** again.

2.  **Update $\text{DATABASES}$ Settings:** Use the credentials and endpoint from your AWS RDS setup.

    ```python
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.postgresql',
            'NAME': 'proshop',                          # AWS Initial database name (e.g., proshop)
            'USER': 'DennisIvy',                        # AWS Master username
            'PASSWORD': 'your_aws_password',            # AWS Master password (BLURRED IN VIDEO)
            'HOST': 'your-rds-endpoint.aws.com',        # PASTE YOUR COPIED AWS ENDPOINT HERE
            'PORT': '5432',                             # Default RDS port
        }
    }
    ```

---

## 6\. Migrate and Populate the Live Database

The live AWS database is empty, so you need to run the migrations again.

1.  **Stop the Django Server:** ($\text{Ctrl}+\text{C}$).
2.  **Run Migrations:** This applies your tables to the **new AWS database**.
    ```bash
    python manage.py migrate
    ```
3.  **Create Superuser (Again):** Create an admin user for the empty live database.
    ```bash
    python manage.py createsuperuser
    ```
4.  **Restart Server and Populate Data:**
    - Run $\text{python manage.py runserver}$.
    - Log in as the new superuser.
    - Add your products/data via the admin panel. This data is now being stored in the **live AWS RDS database**.

Your backend is now running on a production-ready PostgreSQL database hosted on AWS, ready for deployment.

---

### Note:

Django's built-in mechanism for serving static files is designed for **development use only** (when `DEBUG=True`) and is **not efficient or secure** for production environments.

Here are the two main types of third-party modules commonly used in Django for this purpose:

#### 1. **Packages for serving static files directly from the web application**

These libraries integrate into your Django application (via middleware) to serve the static assets (CSS, JS, images, etc.) that you collect with `collectstatic`. This is often a simpler solution for smaller or medium-sized projects, or when deploying to platforms like Heroku.

- **Example:** **WhiteNoise** is the most popular package for this approach. It efficiently serves static files with correct headers for caching and compression.

---

#### 2. **Packages for integrating with cloud/external storage**

These libraries provide **custom file storage backends** for Django, allowing the static files (and especially user-uploaded media files) to be stored and served directly from a cloud service like AWS S3, Google Cloud Storage, or DigitalOcean Spaces. This is the standard practice for larger, high-traffic, or containerized applications as it offloads the serving burden from your application server, provides scalability, and often uses a Content Delivery Network (CDN) for faster global delivery.

- **Example:** **`django-storages`** is a very popular package that provides storage backends for various cloud services (e.g., S3, Azure). You configure it in your Django settings to tell the `collectstatic` command where to upload the files and how to generate the correct public URLs for them.

## Video: Storing product images in AWS S3 bucket

It's a great choice to use **Amazon S3** for storing media files like product images in a Django and DRF application, especially when preparing for production deployment. Django's default setup isn't designed for serving static or media files efficiently in a production environment.

---

### 1\. Understanding the Need for AWS S3

Django, by default, is **not built to store static or media files in production**. While the default setup (using the local file system) works fine during development, a third-party solution is needed for production.

- **Static Files (CSS, JS):** For global static files, the course suggests using a third-party module called **White Noise** (covered in a later video). This serves the core CSS and JavaScript files.
- **Media Files (Product Images):** For dynamic files like product images, **Amazon Web Services (AWS) Simple Storage Service (S3)** is used. S3 is a highly scalable, reliable, and secure object storage service. It's comparable to the **RDS (Relational Database Service)** used previously for the PostgreSQL database.

---

### 2\. Setting Up the S3 Bucket

The first step is to create a storage container, known as a **bucket**, on AWS S3.

#### What is a Bucket?

An S3 **bucket** is essentially a top-level folder where you upload your files. Instead of uploading product images to the local `static/images` folder in your application, you will configure Django to send them directly to this bucket.

#### Bucket Creation Steps

1.  **Create an AWS Account:** If you don't have one, create an account. A payment method is required for billing.
2.  **Navigate to S3:** Search for and select the **S3 service** in the AWS console.
3.  **Create Bucket:**
    - **Name:** Choose a **globally unique** name (e.g., `pro-shop-bucket-demo`). If the name is already taken, you'll need to try another.
    - **Region:** Set this to your preferred region (e.g., US West Oregon).
    - **Public Access Settings:** Crucially, to allow the public internet to view the images on your website, you must **disable** "Block all public access." This is the first step toward making the images accessible.
    - **Acknowledgement:** Acknowledge that turning off the block allows public access. This is a security warning, reminding you **not to store private or sensitive data** (like private PDFs) in this bucket, as anyone with the URL can access the content.
    - **Creation:** Accept the default settings for everything else and create the bucket.

---

### 3\. Configuring Bucket Permissions (Bucket Policy)

After creation, the bucket can store files, but they are not yet accessible via a public URL. Attempting to access an uploaded image's URL will result in an **"Access Denied"** error. You need to set a **Bucket Policy** to define the permissions.

1.  **Go to Permissions:** Navigate to the created bucket and select the **Permissions** tab.
2.  **Set Bucket Policy:** You need to add a JSON-formatted policy that grants public read access to the objects inside.
    - **Policy Content:** This policy typically states that any user can perform the `s3:GetObject` action (read/view) on the bucket's resources.
    - **Resource ARN:** The policy's `Resource` field must be updated to include the specific **bucket name** you created (e.g., `arn:aws:s3:::pro-shop-bucket-demo/*`). The `/*` grants access to all objects within the bucket.
3.  **Save:** Once saved, you will see a warning that the bucket is **Publicly Accessible**. The images can now be viewed via their public S3 URL.

---

### 4\. Creating an IAM User for Application Access

For your Django application to securely upload and manage files in the S3 bucket, it needs a specific set of credentials. This is handled by **AWS Identity and Access Management (IAM)**.

1.  **Navigate to IAM:** Search for and select the **IAM service**.
2.  **Create a User:**
    - **User Name:** Choose a name (e.g., `Dennis-Ivy`).
    - **Access Type:** Select **Programmatic access**. This provides an **Access Key ID** and a **Secret Access Key** that the application will use.
3.  **Set Permissions:**
    - **Attach Existing Policies Directly:** Search for and select **AmazonS3FullAccess**. This grants the user full control over all S3 buckets in your AWS account. _Note: For production, it's best practice to limit this to only the necessary bucket(s), but for a tutorial, full access is often used for simplicity._
4.  **Finish Creation:** The system will display the **Access Key ID** and **Secret Access Key**. **These must be copied immediately**, as the Secret Key will not be shown again. **Crucially, never expose your Secret Access Key in public code.**

---

### 5\. Configuring Django Settings

The application now needs to be configured to use S3 for file storage, connecting with the IAM user credentials.

#### Installation

Two Python packages are required:

1.  **`boto3`**: The official AWS SDK for Python, which handles the connection to AWS services like S3.
2.  **`django-storages`**: A collection of custom storage backends for Django, including one for AWS S3.

Install them using `pip`:

```bash
pip install boto3 django-storages
```

#### `settings.py` Configuration

In your Django project's `settings.py` file, you need to add and update the following settings:

| Setting                     | Value/Configuration                          | Description                                                                                                                                                               |
| :-------------------------- | :------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **INSTALLED_APPS**          | Add `'storages'`                             | Registers the `django-storages` app.                                                                                                                                      |
| **AWS_STORAGE_BUCKET_NAME** | `'pro-shop-bucket-demo'`                     | The unique name of the S3 bucket.                                                                                                                                         |
| **AWS_ACCESS_KEY_ID**       | `'YOUR_IAM_ACCESS_KEY_ID'`                   | The Access Key ID of the IAM user.                                                                                                                                        |
| **AWS_SECRET_ACCESS_KEY**   | `'YOUR_IAM_SECRET_ACCESS_KEY'`               | The Secret Access Key of the IAM user.                                                                                                                                    |
| **DEFAULT_FILE_STORAGE**    | `'storages.backends.s3boto3.S3Boto3Storage'` | Tells Django to use S3 for all file uploads, overriding the default local file storage (`MEDIA_ROOT`).                                                                    |
| **AWS_QUERY_STRING_AUTH**   | `False`                                      | **Optional but recommended.** By default, S3 URLs include extra query string information for authentication. Setting this to `False` creates cleaner, public-facing URLs. |

**Security Note:** The Access Key and Secret Key should ideally be hidden as **environment variables** and not committed directly into `settings.py`.

---

### 6\. Testing and Uploading Images

With the configuration complete, Django will automatically handle the file uploads to S3.

1.  **Initial Problem:** Existing image URLs in the database (which point to the old local file path) will break because Django is now looking for them in the S3 bucket, and they aren't there yet.
2.  **Re-uploading:** To fix this, you must **re-upload** all your product images through the Django admin panel.
    - When you edit a product and upload a new image, Django uses the `S3Boto3Storage` backend to send the file to the S3 bucket.
    - The product's image field in the database is automatically updated to the new, publicly accessible **S3 URL**.
3.  **Verification:**
    - Check your website: The images should now load correctly from the S3 URL.
    - Check the S3 bucket: The newly uploaded image files will be visible in the bucket's "Objects" list.

This setup successfully offloads media storage to a robust, production-ready service, preparing the application for deployment.

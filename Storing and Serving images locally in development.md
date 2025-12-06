# Storing and Serving images locally in development

Storing images locally during development simplifies testing and avoids incurring S3 costs until you deploy.

To store post images locally in your Django development environment, you need to follow these steps:

1.  **Install Pillow**
2.  **Update `settings.py`** (to define where files are stored)
3.  **Update `urls.py`** (to serve the files)
4.  **Update your `Post` model** (to include the image field)

## Step 1: Install Pillow

Django's `ImageField` requires the Pillow library to handle image processing.

```bash
pip install Pillow
```

## Step 2: Update Your Model (`models.py`)

You need to add an `ImageField` to your `Post` model. I will modify the provided file content to include this field.

**`models.py` (Modified):**

```python
from django.db import models
from django.conf import settings
from django.utils.text import slugify

# We reference the custom User model using settings.AUTH_USER_MODEL
User = settings.AUTH_USER_MODEL


class Post(models.Model):
    author = models.ForeignKey(
        User,
        on_delete=models.SET_NULL,
        null=True,
    )

    # Essential post fields
    title = models.CharField(max_length=200, unique=True)
    slug = models.SlugField(max_length=200, blank=True)
    content = models.TextField()
    excerpt = models.CharField(
        max_length=300,
        blank=True,
    )

    # 💥 NEW IMAGE FIELD 💥
    image = models.ImageField(
        upload_to='post_images/',  # Files go into a 'post_images' subfolder inside MEDIA_ROOT
        blank=True,
        null=True,
    )

    # Management fields
    created_at = models.DateTimeField(auto_now_add=True)
    # ... rest of the model ...
```

After modifying the model, run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## Step 3: Configure `settings.py`

You need to tell Django two things: where to physically store uploaded files (`MEDIA_ROOT`) and the URL to access them (`MEDIA_URL`).

Add the following lines, usually at the bottom of your project's `settings.py` file:

```python
# settings.py

# ... other settings ...

# Absolute path to the directory where user-uploaded media files will be stored.
# This creates a 'media/' folder at the root of your project.
MEDIA_ROOT = BASE_DIR / "media"

# The URL prefix for media files (e.g., accessed at http://127.0.0.1:8000/media/...)
MEDIA_URL = "/media/"

# Note: In production (AWS S3), you would change or override these settings.
```

---

## Step 4: Configure URLs (`urls.py`)

By default, Django does not serve user-uploaded files for security reasons. You must explicitly tell Django to serve them **only when `DEBUG=True`** (i.e., in development).

Add the following logic to your **project-level** `urls.py` file (the one that includes the URLs from all your apps):

**`project_name/urls.py` (Modified):**

```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings # Import settings
from django.conf.urls.static import static # Import static function

urlpatterns = [
    path("admin/", admin.site.urls),
    # ... include other app URLs here ...
]

# 💥 SERVE MEDIA FILES ONLY IN DEVELOPMENT 💥
if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

Now, when you run your server (`python manage.py runserver`), any image uploaded through the admin or your API will be saved locally in a `media/post_images/` folder, and you can access it via a URL like `http://127.0.0.1:8000/media/post_images/my_image.jpg`.

## Note:

In this setup, the image field in post objects will be something like this:

```json
"image": "http://localhost:8000/media/post_images/PersianGulf.jpg",
```

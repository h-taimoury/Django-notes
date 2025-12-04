# Image uploading to AWS S3

## Django Project Setup

Handling file uploads and storing them on AWS S3 is a multi-step process that involves configuring Django, installing external libraries, and updating your model and serializer.

Here is a step-by-step guide to integrate an image field into your `Post` model and use AWS S3 for storage.

### Step 1: Install Necessary Libraries

You need four libraries to handle image processing and S3 integration:

```bash
pip install Pillow django-storages boto3
```

- **Pillow:** Required by Django's `ImageField` to handle image validation and manipulation.
- **django-storages:** Provides a way for Django to use custom storage backends, like S3.
- **boto3:** The Amazon Web Services (AWS) SDK for Python, which `django-storages` uses to communicate with S3.

---

### Step 2: Configure AWS S3 in `settings.py`

You must configure your Django project to use S3 as the default file storage system. Add the following to your project's `settings.py` file (replacing placeholders with your actual AWS credentials and bucket name):

```python
# settings.py

# Add 'storages' to your installed apps
INSTALLED_APPS = [
    # ... your existing apps
    'storages',
]

# --- AWS S3 Configuration ---

# 1. AWS Credentials
# IMPORTANT: Use environment variables (os.environ) for production!
AWS_ACCESS_KEY_ID = 'YOUR_ACCESS_KEY_ID'
AWS_SECRET_ACCESS_KEY = 'YOUR_SECRET_ACCESS_KEY'

# 2. Bucket and Region
AWS_STORAGE_BUCKET_NAME = 'your-blog-s3-bucket-name'
AWS_S3_REGION_NAME = 'eu-central-1' # Example: change to your region (e.g., us-east-1)

# 3. Static/Media File Configuration
# Set the default storage backend to S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'

# Optional: Configuration for media files (uploads)
# Files will be uploaded to this S3 folder, e.g., 'https://your-bucket.s3.amazonaws.com/media/...'
AWS_LOCATION = 'media'
MEDIA_URL = f"https://{AWS_STORAGE_BUCKET_NAME}.s3.amazonaws.com/{AWS_LOCATION}/"
```

### Step 3: Update the `Post` Model in `models.py`

Add the `image` field to your `Post` model. We use `ImageField` because it performs image-specific validation and requires Pillow.

**`models.py` (Modified):**

```python
from django.db import models
# ... other imports ...

class Post(models.Model):
    author = models.ForeignKey(
        # ...
    )

    # Essential post fields
    title = models.CharField(max_length=200, unique=True)
    # ... other fields ...
    content = models.TextField()

    # 💥 NEW IMAGE FIELD 💥
    image = models.ImageField(
        upload_to='post_images/', # This is the sub-folder inside your S3 bucket (e.g., media/post_images/)
        blank=True,               # The field is optional
        null=True,                # Allows NULL values in the database
    )

    # Management fields
    created_at = models.DateTimeField(auto_now_add=True)
    # ... rest of the model ...
```

#### Run Migrations

After modifying the model, you must create and apply migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 4: Update the `PostWriteSerializer` in `serializers.py`

Since the image field is new and needs to be accepted during post creation (`POST`), you must add it to your write serializer.

**`serializers.py` (Modified):**

```python
# ... other classes ...

class PostWriteSerializer(serializers.ModelSerializer):
    """
    Serializer used for creating (POST) and updating (PUT/PATCH) a Post.
    """

    url = serializers.SerializerMethodField()

    class Meta:
        model = Post
        # Add 'image' to the fields tuple
        fields = ("title", "content", "excerpt", "is_published", "image", "url")
        read_only_fields = ["url"]

    # ... rest of the serializer ...
```

#### Displaying the Image

You should also add the `image` field to your read serializers (`PostListSerializer` and `PostDetailSerializer`) to ensure the URL for the image is returned in `GET` requests.

The `ImageField` will automatically return the **absolute URL** to the image on S3 when serialized.

**Example `PostListSerializer` Modification:**

```python
class PostListSerializer(serializers.ModelSerializer):
    # ...
    class Meta:
        model = Post
        fields = (
            "id",
            "title",
            "slug",
            "excerpt",
            "created_at",
            "image", # <-- Add here
            # ...
        )
```

---

### Final Step: How it Works

1.  When an admin sends a `POST` request to your ViewSet, they send the image file data as a `multipart/form-data` request.
2.  DRF accepts the file and passes it to the `PostWriteSerializer`.
3.  When `serializer.save()` is called, Django sees that the `Post.image` field has data and is configured with `DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'`.
4.  Django automatically uses the `django-storages` backend to upload the file to your specified AWS S3 bucket, saving the file's S3 URL (relative path) to the `image` column in the database.
5.  On subsequent `GET` requests, the serializer reads the S3 path and combines it with your `MEDIA_URL` (S3 URL) to return the full, publicly accessible image URL.

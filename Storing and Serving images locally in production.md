# Storing and Serving images locally in production

## The Gemini generated this guideline for me but I haven't tested it 

It is **absolutely possible** and is the standard way to host user-uploaded files on shared hosting without S3.

The key difference from your development setup is that you must shift the responsibility of serving the media files from **Django (which is slow and insecure in production)** to the **web server (Apache/Nginx) managed by cPanel.**

Here is the professional workflow for setting up media file storage on a shared host using cPanel.

---

## The Shared Hosting Media Workflow

Your approach will rely on two main concepts:

1.  **Dedicated Directory:** Creating a `media` folder that the web server can access directly.
2.  **Web Server Alias:** Telling the web server (via a directive like `.htaccess`) to intercept requests to your `MEDIA_URL` (e.g., `/media/`) and serve the content directly from that physical folder, bypassing Django entirely.

### 1\. Configure Django `settings.py` (The Same as Dev)

Your `settings.py` configuration should remain identical to your local development setup, defining the public URL and the server's absolute file path:

```python
# settings.py (For Production/Server)

# The public URL prefix (e.g., https://yourdomain.com/media/...)
MEDIA_URL = "/media/"

# The ABSOLUTE path on the server where the files will be physically stored.
# IMPORTANT: This path often needs to be outside your application's source code
# directory to survive code updates. You must determine the full path on your server.
# Example: /home/cpaneluser/public_html/media
MEDIA_ROOT = "path/to/your/media/folder"
```

### 2\. Prepare the Server Directory (cPanel)

You must ensure the `media` folder is accessible by the web server process.

- **Move `media`:** Using the cPanel File Manager, move your `media` folder **outside** of the main application directory, often placing it inside the **`public_html`** folder or at the same level as your application source code.
- **Set Permissions:** Ensure the `media` directory and its subfolders have the correct file permissions (often **755**) so the Python/WSGI process can **write** to it (when a user uploads an image) and the Apache web server can **read** from it (when a user views an image).

### 3\. Configure the Web Server (`.htaccess` File)

Since you cannot directly edit the main Apache or Nginx configuration files on shared hosting, you use the **`.htaccess`** file in your public directory to create a rule (an **Alias**) that tells the web server where to look for media files.

Find the `.htaccess` file that is serving your Django application (it's often in the same directory as your `passenger_wsgi.py` or the entry point). You need to add a rule **before** any rule that passes the request to Django's WSGI application.

```htaccess
# .htaccess (In your application's root directory)

# 1. Turn on the Rewrite Engine
RewriteEngine On

# 2. Add an Alias/RewriteRule for Media Files
# If the request starts with /media/ (your MEDIA_URL)
RewriteRule ^media/(.*)$ /path/to/your/media/folder/$1 [L]

# Note: /path/to/your/media/folder/ must be the exact directory on the server,
# e.g., /home/cpaneluser/public_html/media/

# 3. Add the rule to send all other requests to Django (WSGI/Passenger)
# (Your existing cPanel Django config rule goes here)
# ...
```

### 4\. Remove Development URL Serving

**Crucially**, you must remove the development-only serving code from your project's main `urls.py` file, as the web server is now handling it.

**`project_name/urls.py`:**

```python
# project_name/urls.py

# ...
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... your paths
]

# ❌ REMOVE THIS BLOCK FOR PRODUCTION ❌
# if settings.DEBUG:
#     urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

By completing these steps, you ensure the web server intercepts and serves the media files efficiently, leaving the Django application to handle only API logic.

For a deeper dive into serving files on cPanel, you can watch this video: [Hosting a Django site on cPanel](https://www.google.com/search?q=https://www.youtube.com/watch%3Fv%3DF07gB3vBNSw). This video introduces the main service used for identity and access management in Google Cloud, which is related to managing access to resources like files, similar to how Apache manages access to your media files on the server.

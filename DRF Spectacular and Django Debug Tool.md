# DRF Spectacular and Django Debug Tool

## How to get started with DRF Spectacular

Setting up **`drf-spectacular`** is quick and provides instant, interactive documentation.

Here are the step-by-step instructions for getting started with `drf-spectacular` in your Django project:

### Step 1: Installation

Install the package using pip:

```bash
pip install drf-spectacular
```

---

### Step 2: Configure `settings.py`

Add `drf_spectacular` to your `INSTALLED_APPS` in your project's `settings.py` file:

```python
# settings.py

INSTALLED_APPS = [
    # ... other apps
    'django.contrib.staticfiles',
    # ...
    'rest_framework',
    'drf_spectacular',  # <-- Add this
    'blog', # Your app name
]

# OPTIONAL: Add configuration settings for spectacular
# This is usually not required but can be helpful for branding/metadata
SPECTACULAR_SETTINGS = {
    'TITLE': 'Your Blog API',
    'DESCRIPTION': 'Detailed documentation for the Blog REST API',
    'VERSION': '1.0.0',
    # This setting tells Spectacular where to find the API schema generation logic
    'SERVE_INCLUDE_SCHEMA': False,
}
```

### Step 3: Define URLs in Project `urls.py`

You need to add two main URLs to your **project-level** `urls.py` (the one next to your `settings.py`):

1.  The **Schema View:** Generates the OpenAPI (JSON/YAML) specification file.
2.  The **UI View:** Renders the Swagger UI (the interactive frontend) by reading the schema file.

Open your project's `urls.py` and add the following imports and paths:

```python
# project/urls.py

from django.contrib import admin
from django.urls import path, include
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView # <-- Import these

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.urls')), # Your existing API endpoints

    # 1. API Schema View (The raw JSON/YAML specification file)
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),

    # 2. Swagger UI View (The interactive documentation interface)
    path(
        'api/schema/swagger-ui/',
        SpectacularSwaggerView.as_view(url_name='schema'), # 'url_name' points to the schema view above
        name='swagger-ui'
    ),

    # You can optionally add ReDoc (another documentation style)
    # path('api/schema/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

### Step 4: Run the Server and Access

1.  Make sure your server is running:
    ```bash
    python manage.py runserver
    ```
2.  Open your browser and navigate to the new Swagger UI URL:
    ```
    http://localhost:8000/api/schema/swagger-ui/
    ```

You should now see the interactive documentation interface, which lists all your Post and Comment endpoints, along with the expected fields from your serializers. You can use the **"Try it out"** button to execute requests against your live backend.

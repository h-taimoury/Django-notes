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

Add `drf_spectacular` to your `INSTALLED_APPS` in your project's `settings.py` file and also register spectacular AutoSchema with DRF:

```python
# settings.py

INSTALLED_APPS = [
    # ... other apps
    'django.contrib.staticfiles',
    # ...
    'rest_framework',
    'drf_spectacular',  # <-- Add this
    'users',
    'posts',
]

REST_FRAMEWORK = {
    # YOUR SETTINGS
    # Add this 👇:
    # The following line is for drf_spectacular library to work
    "DEFAULT_SCHEMA_CLASS": "drf_spectacular.openapi.AutoSchema",
}

# OPTIONAL: Add configuration settings for spectacular
# This is usually not required but can be helpful for branding/metadata
SPECTACULAR_SETTINGS = {
    'TITLE': 'My Blog API',
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
# The following import is here for drf_spectacular library
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('blog.urls')), # Your existing API endpoints

    # The following two URL patterns are here for drf_spectacular library
    # API Schema View (The raw JSON/YAML specification file)
    path("api/schema/", SpectacularAPIView.as_view(), name="schema"),
    # Swagger UI View (The interactive documentation interface)
    path(
        "api/schema/swagger-ui/",
        SpectacularSwaggerView.as_view(url_name="schema"),
        name="swagger-ui",
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

## Note:

In DRF Spectacular UI, when using Authorize button, we shouldn't past the access token alongside a `Bearer ` string, just past the access token in the field alone. DRF Spectacular adds `Bearer ` string itself.

### Problems I faced with DRF Spectacular and the solutions I found

## Problem:

When using FBVs, DRF Browsable API and DRF Spectacular didn't generate html forms for API endpoints which handled POST/PUT/PATCH requests (the methods that need a form) alongside a GET request (the method that doesn't need a form).

They both worked fine for any API endpoint that handles GET requests and any API endpoint that only handles POST requests (no handling of GET requests). So note that DRF Browsable API and DRF Spectacular both have this problem.

## Solution:

I haven't found a solution yet.

## Problem:

When using CBVs, DRF Spectacular (not DRF Browsable API, the problem remained the same with DRF Browsable API) did generate html forms for API endpoints which handled POST/PUT/PATCH requests (the methods that need a form) alongside GET requests (the method that doesn't need a form), but the Execute button didn't work. The Execute button did work for GET requests but didn't work for requests needing a form. I could send these requests using application/json request bodies, but not using multipart/form-data request bodies.

## Solution:

I found a solution in StackOverFlow. The person who asked the question had exactly the same problem as me (I check his provided code) and someone had answered this:

`Add this option your SPECTACULAR_SETTINGS:`

```python
'COMPONENT_SPLIT_REQUEST': True,
```

Here is more data about this setting from DRF Spectacular documentation:

From [Settings](https://drf-spectacular.readthedocs.io/en/latest/settings.html) section in DRF Spectacular documentation which indicates the default value for this setting and explains about it:

```python
# Split components into request and response parts where appropriate
# This setting is highly recommended to achieve the most accurate API
# description, however it comes at the cost of having more components.
'COMPONENT_SPLIT_REQUEST': False,
```

From [Client generation](https://drf-spectacular.readthedocs.io/en/latest/client_generation.html) section in DRF Spectacular documentation:

### Client generation

drf-spectacular aims to generate the most accurate schema possible under the constraints of OpenAPI 3.0.3. Unfortunately, sometimes this goal conflicts with generating a good and functional client.

To serve the two main use cases, i.e. documenting the API and generating clients, we opt for getting the most accurate schema first, and then provide settings that allow to resolve potential issues with client generation.

Note

TL;DR - Simply setting 'COMPONENT_SPLIT_REQUEST': True will most likely yield the best and most accurate client.

Note

drf-spectacular generates warnings where it recognizes potential problems. Some warnings are important to having a correct client. Fixing all warning is highly recommended.

Note

For generating clients with CI, we highly recommend using ./manage.py spectacular --file schema.yaml --validate --fail-on-warn to catch potential problems early on.

#### Component issues

Most client issues revolve around the construction of components. Some client targets have trouble with readOnly and required fields like id. Even though technically correct, the generated code may not allow creating objects with id missing for POST requests. Some fields like FileField behave very differently on requests and responses and are simply not translatable into a single component.

The most useful setting is 'COMPONENT_SPLIT_REQUEST': True, where all affected components are split into request and response components. This takes care of almost all required, writeOnly, readOnly issues, and generally delivers code that is easier to understand and harder to misuse.

Sometimes you may only want to fix the required/readOnly issue without splitting all components. This can be explicitly addressed with 'COMPONENT_NO_READ_ONLY_REQUIRED': True. Because this setting waters down the correctness of the schema, we generally recommend using COMPONENT_SPLIT_REQUEST instead.

'COMPONENT_SPLIT_PATCH': True is already enabled by default as PATCH and POST requests clash on the required property and cannot be adequately modeled with a single component.

Relevant settings:

```python
# Split components into request and response parts where appropriate

'COMPONENT_SPLIT_REQUEST': False,

# Aid client generator targets that have trouble with read-only properties.

'COMPONENT_NO_READ_ONLY_REQUIRED': False,

# Create separate components for PATCH endpoints (without required list)

'COMPONENT_SPLIT_PATCH': True,
```

# Advanced Search in PostgreSQL

Leveraging PostgreSQL extensions for advanced search is often far superior to relying on simple string lookups. **Trigram** and **Unaccent** are two of the most popular for boosting search quality.

Here is a breakdown of how to integrate and use them in a Django project.

---

## Adding PostgreSQL Extensions to Your Project

Before you can use the functions provided by these extensions, you need to enable them in two places: the **database** and your **Django project**.

### 1\. Enable in Django Settings

You must tell Django that your project will be using PostgreSQL-specific features by adding `django.contrib.postgres` to your `INSTALLED_APPS`.

```python
# settings.py

INSTALLED_APPS = [
    # ... your other apps
    'django.contrib.postgres', # ⬅️ Essential for using the extensions and their lookups
]
```

### 2\. Enable in the Database (Migration)

You must run a migration to instruct PostgreSQL to install the actual extension software. This uses Django's `TrigramExtension` and `UnaccentExtension` classes.

- **Create a new empty migration:**
  ```bash
  python manage.py makemigrations --empty <your_app_name>
  ```
- **Edit the new migration file (e.g., `0002_enable_extensions.py`):**

<!-- end list -->

```python
# 0002_enable_extensions.py

from django.contrib.postgres.operations import (
    TrigramExtension,
    UnaccentExtension,
)
from django.db import migrations

class Migration(migrations.Migration):
    dependencies = [
        ('your_app_name', '0001_initial'),
    ]

    operations = [
        # 1. Enables the 'pg_trgm' extension in the database
        TrigramExtension(),
        # 2. Enables the 'unaccent' extension in the database
        UnaccentExtension(),
    ]
```

- **Run the migration:**
  ```bash
  python manage.py migrate
  ```

---

## Most Important Extensions for Search

While PostgreSQL offers hundreds of extensions, these three are the most important for comprehensive text search in Django:

| Extension Name | Django Class        | Function                       | Purpose                                                                                             |
| :------------- | :------------------ | :----------------------------- | :-------------------------------------------------------------------------------------------------- |
| **`pg_trgm`**  | `TrigramExtension`  | **Fuzzy Search / Similarity.** | Allows measuring the similarity between strings, essential for typo-tolerant search.                |
| **`unaccent`** | `UnaccentExtension` | **Accent-Insensitive Search.** | Removes diacritics (accents like é, à, ü) from text to match across languages.                      |
| **`pg_tsm`**   | (No direct class)   | **Full-Text Search (FTS).**    | The standard for fast, ranked, word-based search (uses `SearchVector` and `SearchQuery` in Django). |

---

## Examples of `TrigramExtension` and `UnaccentExtension`

Once the extensions are enabled, Django exposes custom lookups and functions you can use in your QuerySets.

### 1\. `TrigramExtension` (Fuzzy Search/Typo Tolerance)

Trigrams break strings into sequences of three characters (e.g., "Django" -\> " D", "Dja", "jan", "ang", "ngo", "go "). Similarity is determined by how many trigrams two strings share.

#### A. Similarity Lookup (`__trigram_similar`)

This is used for fuzzy matching.

**Goal:** Find product titles that are similar to the misspelled term "Choclate".

```python
# Requires importing the TrigramSimilarity function or using the lookup
from django.contrib.postgres.search import TrigramSimilarity

# 1. Filter using the lookup (simpler)
# Finds titles that share a high degree of trigram similarity with 'Choclate'
similar_products = Product.objects.filter(
    title__trigram_similar='Choclate'
)

# 2. Annotate and Order (More precise for ranking)
# Annotates each product with a similarity score (0.0 to 1.0) and orders by it.
ranked_products = Product.objects.annotate(
    similarity=TrigramSimilarity('title', 'Choclate')
).order_by('-similarity')
```

### 2\. `UnaccentExtension` (Accent-Insensitive Search)

This extension provides the `unaccent` lookup, which removes accents, making "résumé" match "resume."

#### A. Accent-Insensitive Filtering

You typically use the `unaccent` function on both the database field and the search term to ensure a reliable comparison.

**Goal:** Find customers whose name matches 'Renée' even if the user typed 'Renee'.

```python
# Simple accent-insensitive lookup
City.objects.filter(name__unaccent="México")
# Conceptual SQL: WHERE unaccent(name) = 'México'
```

```python
# Chaining unaccent with a standard lookup (e.g., __startswith)
User.objects.filter(first_name__unaccent__startswith="Jerem")
# This finds 'Jeremy', 'Jérémy', 'Jérémie', etc.
```

This is a simple yet powerful way to support multiple languages without complex manual character replacements.

---

# How to set up a Django project:

## We need Python installed on our machine

So visit python.org and download and install the latest version or just install Anaconda. By installing Anaconda, we get Python and other most used Python libraries.
After that check python being installed by

```
py --version
```

## Create two nested folders:

Create a folder for the project (e.g. Blog) and create a folder for our backend inside it. Let's name it 'backend' and cd into it.

## Create a virtual environment

Use venv module, which is a part of Python's standard library, to create a virtual environment

```
py -m venv .venv
```

This command will create a '.venv' folder inside of the 'backend' folder.

The main external package that provides similar functionality is virtualenv.

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

Every library we need for our project must be installed in the virtual environment. So when installing a package for the project, remember to activate the virtual environment first.

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

Many people use the name `config` for the `Project Package` and many people have struggled with the same issue we went through. I think the Django team made a mistake with the below command many years ago and while it's a very confusing folder structure, they can't get out of it.

Note: When we create a Django project, Django writes some comments in some modules. In one of these modules the comments say something like "Hello from 'Project_Package_Name' project". So Django team considers the name you provide for project package to be your project's name, but obviously this can cause a lot of confusion. So people avoid it and many use 'config' name for project package.

```
django-admin startproject config .
```

Here 'config' is the name of " Project Package" (containing a settings.py and other files). It's the container for all of your project's main settings and configurations.

Let's always use the name "config" for Project Package until we find a better one because many developers do this.

In many tutorials, they use confusing names for it like 'my_project' or 'my_django_project'. It's because Django team seems to prefer this convention of naming project package as your project's name. But it's confusing because it's not the project folder, it's project package folder. Even in Django documentation, a name of 'mysite' has been used which is not good.

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

The next step is to configure the root URLconf in project package to include the URLconf defined in `posts/urls.py`, so navigate to project package folder and open `urls.py` there. Add a path to urlPatterns.

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

## Apply initial migrations

We need to apply initial migrations for the the defualt user model to make yellow warnings in the terminal logs go away. In project directory, run:

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

Navigate to your app's folder and open `models.py` file. This is where we define our models.

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

## Install pillow library

To use `models.ImageField`, we need to install `Pillow` which is an image processing library. So while in venv, run:

```
pip install pillow
```

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

After adding all the models to this file, our `admin.py` file content would be something like above.

Now if we visit admin panel, we will see our app and our 'Products' table there and we can add a product.

## Visit admin panel

We've done these steps by now:

- Defined our models
- Made migrations
- Applied migrations
- Registered our models to admin panel

So now we can visit admin panel and create and manage instances of our models.

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

So in app folder, create a file named `serializers.py` and put this code inside:

```python
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

By `many=True` argument we are telling Django that we want to serialize many objects. In other words, we have a list of objects we want to return not just one object.

In `getProduct` view, we'll set it to False because there, we just want to return one object. Define `getProduct` view like this:

```python
@api_view(['GET'])
def getProduct(request, pk):
    product = Product.objects.get(_id=pk)
    serializer = ProductSerializer(product, many=False)
    return Response(serializer.data)
```

## Authentication and Authorization

Because we're using DRF, handling authentication and authorization is a bit different than using Django built-in authentication and authorization system.

You can read more about authentication in DRF in their website. From `API Guide` dropdown menu, click on `Authentication` and specially search for `token-based authentication`.

In this project, we want to use token-based authentication using a third party package called `Simple JWT`.

Simple JWT is a JSON Web Token authentication plugin for the Django REST Framework.

For full documentation, visit https://django-rest-framework-simplejwt.readthedocs.io/en/latest/.

To install the package, cd into project directory and run:

```
pip install djangorestframework-simplejwt
```

### Project Configuration

Then, your django project must be configured to use the library. In settings.py, add this:

```python
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

Then in users app url config file, include routes for Simple JWT’s `TokenObtainPairView` and `TokenRefreshView` views (these two are class-based views):

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

It is generated because of `DRF Browsable API` feature.

The Django REST Framework (DRF) Browsable API is a feature that automatically generates a user-friendly HTML page when you view your API endpoints in a web browser.

In simple terms, it's a developer tool that turns your raw, hard-to-read JSON or XML API responses into an interactive web page that you can test and explore.

DRF endpoints are browsable: DRF's default configuration makes API endpoints accessible and interactive when viewed in a web browser, unlike most standard REST APIs that only return raw data.

Interaction: This interface isn't just for viewing data; it includes auto-generated HTML forms for endpoints that accept input (like POST or PUT), allowing developers to interact with and test the API without relying on external tools like `Postman`, `Insomnia`, or custom client-side code.

It is a core design philosophy of DRF to make API development and testing as easy and visual as possible.

## Simple JWT Authentication and Customization

### 1\. Generating the Initial Token

The primary purpose of the `/api/users/login/` route is to authenticate a user and generate a JWT pair.

If I submit my user credentials (username and password) to this endpoint and hit **POST**, the system handles the authentication. If successful, it returns an **access token** and a **refresh token**. This is the token we will store on the client-side (e.g., in Local Storage) and use to authenticate subsequent requests to protected API routes by using this token in request's `Authorization` header.

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

**Testing the Change:**

1.  After generating a new token (using the same `POST` request), the previously generated token's expiration date will remain the same.
2.  However, the new token we just generated will now reflect the change.
3.  When pasting the new token into **jwt.io**, the expiration date will be set for **30 days** from now, confirming the customization.

### 2\. Customizing Token Claims and API response

### 1\. Encoding Custom Claims into the JWT

By default, the Simple JWT only encodes the user_id and standard claims (like expiration time). We can add custom data. Visit DRF Simple JWT website and `Customizing token claims` section using this [link](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/customizing_token_claims.html).

**From SimpleJWT Documentation:**

If you wish to customize the claims contained in web tokens which are generated by the `TokenObtainPairView` and `TokenObtainSlidingView` views, create a subclass for the desired view as well as a subclass for its corresponding serializer. Here’s an example of how to customize the claims in tokens generated by the TokenObtainPairView.

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

   But we don't need it (it's from the older documentation and we will discuss it in option 2. It's like they havn't bothered to remove this line!).

2. **Create a Custom Serializer:** Copy and past this code in your `simple_jwt.py` file

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
   Navigate to `settings.py` file in project package and override `TOKEN_OBTAIN_SERIALIZER` like this:

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

Note that the example above will cause the customized claims to be present in both refresh and access tokens which are generated by the view. This follows from the fact that the `get_token` method above produces the `refresh token` for the view, which is in turn used to generate the view’s `access token`.

**Testing the Custom Claims**

After generating a new token, when you paste the **access token** into [jwt.io](http://jwt.io), the decoded payload now includes your custom fields: `username` and `message`. This demonstrates how to encode any user information directly into the token.

---

### Implementing option 2

Note: `Customizing token claims` documentation I see now is different from what tutor displays in the video.

In the video, the documentation (and ofcourse the tutor) creates a custom view too. So create a `simple_jwt.py` file in users app or any other prefered folder and past this code inside it:

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

To simplify frontend state management, we can modify the response to immediately return core user data. This avoids the need for an extra API call or client-side token decoding.

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

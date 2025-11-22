# Create an app in Django project

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

```
def welcome(request):
    return JsonResponse("Hello world from /api/posts", safe=False)
```

## Create app's URL configuration file

To access the view in a browser, we need to map it to a URL - and for this we need to define a URL configuration, or `URLconf` for short. These URL configurations are defined inside each Django app, and they are Python files named `urls.py`.
In new app's folder, create a 'urls.py' file. This file is in charge of routing urls to views.

```
urlpatterns = [
    path('', views.welcome , name='welcome')
]
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

---
title: "Backend Python:Django Rest Framework Project Setup Template"
date: 2018-09-07T16:18:09+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---

I am assuming that pipenv is already installed. Also, assuming that Python 3.5+ already installed in the base environment. 
1. create new environment: `pipenv shell`
1. `pipenv install django djangorestframework markdown django-filter pygments flake8 httpie pytest pytest-django pytest-factoryboy pytest-cov django-extensions djangorestframework-simplejwt django-cors-headers mypy djangorestframework-stubs`
1. `pipenv install --dev --pre black`
1. `create django project:django-admin.py startproject my_proj`
1. `cd my_proj`
1. `python manage.py startapp my_app`

The above is a basic boiler plate.

Some explanation of what got installed:

1. `django djangorestframework markdown django-filter pygments` are recommended essentials from Django rest framework installation guide [ here ](https://www.django-rest-framework.org/#installation)
1. `httpie` for making command-line http requests
1. `flake8` for linting
1. `pytest pytest-django pytest-factoryboy` for testing.
1. `django-extensions` really cool extension. Offers lots of goodies. I mainly use it for `shell_plus` to load all the models whenever I start shell. Real timesaver when developing.
1. `djangorestframework-simplejwt` for jwt based token authentication. Really cool and much better than vanilla token authentication since it does not require database persistance/lookup of tokens.
1. `django-cors-headers` for cors setup.
1. `mypy djangorestframework-stubs` for type checking. In case of a __Django project__, use `django-stubs`
1. `pipenv install --dev --pre black` for code formatting. –pre is needed because black is still in beta.

## Setting up django-extensions
Easy-peasy. `my_proj/my_proj.settings.py` -> add `'django-extensions'` to INSTALLED_APPS

## Setting up mypy
You can write mypy setting in either setup.cfg file or `mypy.ini` file. I prefer `mypy.ini`. This file is at the project root (same level as `manage.py`)

Since my project is a Django rest framework project, I would need to add both `django-stubs` and `djangorestframework-stubs`. My `mypy.ini` file looks like this:

```
[mypy]
plugins =
    mypy_django_plugin.main,
    mypy_drf_plugin.main

ignore_missing_imports = True
warn_unused_ignores = True
strict_optional = True
check_untyped_defs = True
follow_imports = silent
show_column_numbers = True
disallow_any_generics = True
disallow_untyped_calls = True
disallow_untyped_decorators = True
ignore_errors = False
implicit_reexport = False
strict_equality = True
no_implicit_optional = True
warn_redundant_casts = True
warn_unused_configs = True
warn_unreachable = True
warn_no_return = True

[mypy.plugins.django-stubs]
django_settings_module = 'todo.settings'

[mypy-*.migrations.*]
ignore_errors = True
```

Note that following two lines are essential for mypy to work:
```
[mypy.plugins.django-stubs]
django_settings_module = 'todo.settings'
[mypy-*.migrations.*]
ignore_errors = True
```
First one tells mypy where the settings file is for the project. Replace `todo.settings` with `your_project.settings`

The second one is specifically meant to ignore type checking in migrations folder.

Also, this line `disallow_any_generics = False` can sometimes cause mypy errors especially in models. For example, if you have this line in your custom user model,
`class UserManager(BaseUserManager):`, BaseUserManager is a generic type. This will cause mypy type error if `disallow_any_generics=True`. In this scenario, put this to False.

Further more, in all of my test files, `manage.py` and `settings.py` file, I add this on top:
# mypy: ignore-errors
Any file that has this line on top, will be ignored by mypy for type check.

## Setting up pytest
We will setup the skeleton for testing first.

1. create a directory 'tests' in my_proj (inside same folder as `manage.py`)
1. cd tests
1. touch __init__.py
1. mkdir my_app
1. touch __init__.py
1. `touch test_views.py` (in this file we will do integration tests, testing views in `my_app/views.py`)
1. come back to tests folder and create a file called factories.py here. In this file, we will setup all factory objects to create fake data for testing

Note that DRF has its own `tests.py` file in every app we create but with pytest, we don’t use this file. We setup all our tests inside tests folder we setup in step 1. tests.py is used when we use the DRF provided testing mechanism which is sort of based on Python unittest framework. pytest is a more advanced framework also built on top of unittest.

## Setting up JSON Web Token authentication (JWT)
Goto my_proj/my_proj/settings.py file and add the following at the bottom:
```python
REST_FRAMEWORK = {
'DEFAULT_AUTHENTICATION_CLASSES': ('rest_framework_simplejwt.authentication.JWTAuthentication',)
}
```

Then goto my_proj/urls.py file add these:
```python
from rest_framework_simplejwt.views import TokenObtainPairView, TokenRefreshView

urlpatterns = [
...,
path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
...,
]
```

That’s it. jwt authentication is setup. We use this to guard specific view methods like this..

Suppose there is an endpoint called ProductsList that lists all products and we want to guard this so that only admin users can see all the products.
```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.permissions import IsAuthenticated

from .serializers import ProductSerializer
from .models import Product

class ProductsList(APIView):
    permission_classes = (IsAuthenticated,)
    
    def get(self, request, format=None):
        try:
            products = Product.objects.all()
            serializer = ProductSerializer(products, many=True)
            return Response(serializer.data)
        except Exception:
            return Response(status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```
In the above code, these two lines implement the guards:

`from rest_framework.permissions import IsAuthenticated`
`permission_classes = (IsAuthenticated,)`

Note that `IsAuthenticated` class is only useful to check if user is admin or not. If you want to setup object level permissions (as in what tables a user can see) or row-object level permissions (as in what rows in a specific table a user can see), you will need something like `django-guardian`.

Also, we need to setup a superuser using `python manage.py createsuperuser` to actually get the token. To see how to exactly use tokens as well as customize tokens see [ here ](https://github.com/davesque/django-rest-framework-simplejwt#Usage).


## CORS setup
Add this to my_proj/my_proj/settings.py file:
```python
MIDDLEWARE = [
'corsheaders.middleware.CorsMiddleware',
...
]
```

As you see, 'corsheaders.middleware.CorsMiddleware' being the first entry in the list is intentional. This is to ensure that CORS header check is done early on so as to avoid some other middleware intercepting the request and rejecting it due to improper CORS before our CORSMiddleware has had a chance to setup proper CORS headers.
Then in the same settings.py file, add these entries too:
```python
CORS_ORIGIN_ALLOW_ALL = False
CORS_ALLOW_CREDENTIALS = True
CORS_ORIGIN_WHITELIST = [
# TODO - set this properly for production
#'http://127.0.0.1:8080',
#'http://127.0.0.1:8000',
]
CORS_ORIGIN_REGEX_WHITELIST = [
#'http://localhost:8000',
]
```

You will setup CORS_ORIGIN_WHITELIST and CORS_ORIGIN_REGEX_WHITELIST with proper values before putting the site in production

OK, so now that everything is installed, lets get rolling.

## Optional
DRF does not need placeholders for static files like css and js but I anyway do the configuration for these as well. Simply out of habit. If your project is Django only, you will most likely need it. Note that even if we are hosting static files in CDN like AWS, its still good to set them up in ‘settings.py’ since then we can use django management commands to collect all project wide static files into a known directory and then dump this directory into the CDN and reference it in front-end client code.

Firstly, the default settings.py file has `STATIC_URL = '/static/'`
Here ‘/static/’ is just a prefix. You can leave it so. You can even have it set to None. You will need to check this is production. More on this after we have discussed STATIC_ROOT.

Here is tho code snippet of my static file setup.
```python
STATIC_URL = '/static/'
STATICFILES_DIRS = [
    os.path.join(BASE_DIR, 'local_dev_static'),
]
STATIC_ROOT = os.path.join(os.path.dirname(BASE_DIR), 'static_cdn', 'static_root')
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(os.path.dirname(BASE_DIR), 'static_cdn', 'media_root')
```
When it comes to static files, we have multiple places to put them when developing. For app specific static files, the Django recommended way is to make a ‘static’ folder in the app and put all content there. For non-app specific static content, we have STATICFILES_DIRS. I name it ‘local_dev_static’ to make it clear for anyone eyeballing the hierarchy that this is only for local static file storage. Meaning, in production, the static files are not served from here. During development, if we have non-app specific staic files, we can dump them all here. Then call `python manage.py collectstatic` to collect the static files from STATICFILES_DIRS and all ‘static’ folders inside the apps and dump them in STATIC_ROOT.

STATIC_ROOT is the absolute path to the directory where `python manage.py collectstatic` will collect static files for deployment. Note this is might not be the place from where the server will serve them. This is just a convenience location to use django management commands to collect all static files in one directory. Then you can put all the STATIC_ROOT contents into the directory that actually serves them. Note this step is manual. As in you will have to copy paste these files from STATIC_ROOT to location indicated by STATIC_URL (you can of course write your automation but it is outside of Django).

When STATIC_URL is ‘/static/’ then the web server (Apache/Nginx) will automatically prepend the path to mean ‘http://example.com/static/’ where ‘http://example.com’ is your domain. If you decide to actually serve your static files from a sub-domain, like ‘http://static.example.com/’, you need to do this: STATIC_URL = ‘http://static.example.com/’. Of course, you will need to copy-paste all the collected files from STATIC_ROOT to location pointed to by the subdomain in the server.

If STATIC_URL is not set to None, you can use it in your templates like so:
```html
<link rel="stylesheet" href="{{ STATIC_URL }}css/base.css" type="text/css" />
```
and this will be treated as (in this case STATIC_URL is the subdomain):
```html
<link rel="stylesheet" href="http://static.example.com/css/base.css" type="text/css" />
```
It is recommended to use STATIC_URL and not hard code URLs in templates, even though STATIC_URL can be set to None.

Finally, MEDIA_ROOT and MEDIA_URL. These are on the same lines as STATIC_ROOT and STATIC_URL. Media files are anything we upload.

Since I want all STATIC and MEDIA files in one place, I create a directory called ‘static_cdn’. Again, this is just a convenience location. I chose it at the same level as the parent folder of the project. In this directory we have STATIC_ROOT and MEDIA_ROOT. Then we can dump all of its contents to a CDN. In templates, use MEDIA_URL for referencing media files and STATIC_URL for referencing static files.

Please note that in production, Django plays no role in serving static files. Serving static files is totally handled by the server (Apache or Nginx). You can even put all the static files in a CDN like AWS.

Then we also need to setup urlpatterns to serve the media files when using development server. Do it like so:

First include these two imports in urls.py:
```python
from django.conf import settings
from django.conf.urls.static import static
```
then add the urlspatterns for MEDIA like so:
```python
urlpatterns = [
    url(r'^products/(?P<pk>\d+)', EcomProductDetailView.as_view()),
    url(r'^products', EcomProductListView.as_view()),
    url(r'^admin/', admin.site.urls),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```
Note that adding the MEDIA urlpatterns this way, we do not have to explicitly check if we are in DEBUG mode or not as Django will ensure this is only used in Debug mode.
[ Here ](https://docs.djangoproject.com/en/2.1/howto/static-files/#serving-files-uploaded-by-a-user-during-development) is some documentation from Django 2.1.

Prior to Django 1.7, it was done like so:
```python
from django.conf import settings

# ... your normal urlpatterns here

if settings.DEBUG:
    # static files (images, css, javascript, etc.)
    urlpatterns += patterns('',
        (r'^media/(?P<path>.*)$', 'django.views.static.serve', {
        'document_root': settings.MEDIA_ROOT}))
```
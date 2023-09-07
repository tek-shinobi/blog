---
title: "Backend Python:Setting Active Navbar Link in Django Template"
date: 2018-09-07T11:47:47+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
Here is probably the best way to set the active navbar link in Django template. Note, __this needs no jQuery/javascript__.

## Step 1
Create named urls:

```python
from django.conf.urls import url, include
from django.contrib import admin

from django.conf import settings
from django.conf.urls.static import static

from .views import (
    HomeView,
    AboutView,
    ContactView,
    LoginView,
    RegistrationView
)

urlpatterns = [
    #url(r'^products/(?P<slug>[\w-]+)/$', EcomProductDetailSlugView.as_view()),
    url(r'^$', HomeView.as_view(), name='home'),
    url(r'^about/$', AboutView.as_view(), name='about'),
    url(r'^contact/$', ContactView.as_view(), name='contact'),
    url(r'^login/$', LoginView.as_view(), name='login'),
    url(r'^register/$', RegistrationView.as_view(), name='register'),
    url(r'^products/', include('ecom_product.urls', namespace='ecom_product')),
    url(r'^admin/', admin.site.urls),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

```
## Step 2
Create a Navbar template. I keep it stored in a file called navbar.html inside “templates/base” folder. This “templates” folder is at same level as manage.py
I am using bootstrap 4 navbar component.

```html
{% url 'home' as home_url %}
{% url 'contact' as contact_url %}
{% url 'ecom_product:list' as ecomproduct_url %}
{% url 'login' as login_url %}
{% url 'register' as register_url %}

<nav class="navbar navbar-expand-lg navbar-light bg-light">
    <a class="navbar-brand" href="#">
        {% if brand_name %} {{brand_name}} {% else %} Default {% endif %}
    </a>
    <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
        <span class="navbar-toggler-icon"></span>
    </button>
    <div class="collapse navbar-collapse" id="navbarNav">
        <ul class="navbar-nav">
            <li class="nav-item {% if request.path == home_url %} active {%endif%}">
                <a class="nav-link" href="{{home_url}}">Home <span class="sr-only">(current)</span></a>
            </li>
            <li class="nav-item {% if request.path == contact_url %} active {%endif%}">
                <a class="nav-link" href="{{contact_url}}">Contact</a>
            </li>
            <li class="nav-item {% if request.path == ecomproduct_url %} active {%endif%}">
                <a class="nav-link" href="{{ecomproduct_url}}">Products</a>
            </li>
            {% if request.user.is_authenticated %}
            <li class="nav-item {% if request.path == login_url %} active {%endif%}">
                <a class="nav-link" href="{{login_url}}">Logout</a>
            </li>
            {% else %}
            <li class="nav-item {% if request.path == login_url %} active {%endif%}">
                <a class="nav-link" href="{{login_url}}">Login</a>
            </li>
            <li class="nav-item {% if request.path == register_url %} active {%endif%}">
                <a class="nav-link" href="{{register_url}}">Register</a>
            </li>
            {% endif %}
        </ul>
    </div>
</nav>
```
In bootstrap 4, “active” class is put on the currently active link. This is the best way to do it using Django templates:
`{% if request.path == ecomproduct_url %} active {%endif%}`
This will highlight `Products` in navbar. Note, this way also needs you to first create a variable for the named url like so
`{% url 'ecom_product:list' as ecomproduct_url %}`

## Step 3
Include the navbar like so:
```html
{% extends 'base.html' %}
{% load static %}

{% block main_css %}
  <link rel="stylesheet" href="{% static 'css/main.css' %}">
{% endblock %}

{% block content%}
{% include 'base/navbar.html' with brand_name=brand_name %}
<form method="POST">{% csrf_token %}
    {{form}}
    <button class="btn" type="submit">Submit</button>
</form>
{% endblock %}
```


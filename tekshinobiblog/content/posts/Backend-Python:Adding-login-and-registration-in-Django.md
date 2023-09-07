---
title: "Backend Python:Adding Login and Registration in Django"
date: 2018-09-07T13:47:36+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---

Adding login and registration is very straight forward in Django. Note that Django provides a built in User model.

User model exists ONLY for authentication. Use the User model to only store absolutely necessary info for authentication. Like Username and password (or if using custom user model, email and password.. I usually make the email as the unique field). Nothing else. Anything else, like gender, blah blah blah goes in other models like profile etc.

Please see post titled `Adding Custom User Model in Django
` on how to make a custom User model.

__NEVER__ access `User` model directly. Use the `get_user_model()` method provided by Django. This is so that we keep everything DRY and only change the User model in settings (if we have to).

Three simple steps:

1. Make the Django forms for login and registration
1. Implement the view logic for login and registration
1. Do the wiring in `urls.py`

For Django forms, I have created a `forms.py` file at the same level as `settings.py`

```python
from django import forms
from django.contrib.auth import get_user_model

class ContactForm(forms.Form):
    fullname = forms.CharField(
        widget=forms.TextInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Name',
                'name': 'fullname'
            }
        )
    )
    email = forms.EmailField(
        widget=forms.EmailInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Email',
                'name': 'email'
            }
        )
    )
    content = forms.CharField(
        widget=forms.Textarea(
            attrs={
                'class': 'form-control',
                'placeholder': 'Content',
                'name': 'content'
            }
        )
    )

    def clean_email(self):
        email = self.cleaned_data.get('email', None)

        if not email or 'gmail.com' not in email:
            raise forms.ValidationError('email must contain gmail.com')
        return email


class LoginForm(forms.Form):
    username = forms.CharField(
        widget=forms.TextInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Username',
                'name': 'username'
            }
        )
    )

    password = forms.CharField(
        widget=forms.PasswordInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Password',
                'name': 'password'
            }
        )
    )


class RegistationForm(forms.Form):
    username = forms.CharField(
        widget=forms.TextInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Username',
                'name': 'username'
            }
        )
    )

    password = forms.CharField(
        widget=forms.PasswordInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Password',
                'name': 'password'
            }
        )
    )

    password2 = forms.CharField(
        label='Confirm Password',
        widget=forms.PasswordInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Password',
                'name': 'password'
            }
        )
    )

    email = forms.EmailField(
        widget=forms.EmailInput(
            attrs={
                'class': 'form-control',
                'placeholder': 'Email',
                'name': 'email',
            }
        )
    )

    def clean_username(self):
        username = self.cleaned_data.get('username', None)
        User = get_user_model()
        qs = User.objects.filter(username=username)
        if qs.exists():
            raise forms.ValidationError('Username is taken')
        return username

    def clean_email(self):
        email = self.cleaned_data.get('email', None)
        User = get_user_model()
        qs = User.objects.filter(email=email)
        if qs.exists():
            raise forms.ValidationError('email exists')
        return email

    def clean(self):
        data = self.cleaned_data
        password = data.get('password', None)
        password2 = data.get('password2', None)
        email = data.get('email', None)
        if not email or 'gmail.com' not in email:
            raise forms.ValidationError('email must contain gmail.com')        
        if password != password2:
            raise forms.ValidationError('Passwords must match')
        return data
```
Now comes the view logic in views.py
```python
from django.views.generic import TemplateView
from django.contrib.auth import login, authenticate, get_user_model
from django.http import HttpResponseRedirect

from .forms import (
    ContactForm,
    LoginForm,
    RegistationForm
)

class HomeView(TemplateView):
    http_method_names = ['get']
    template_name = "home_page.html"

    def get_context_data(self, *args, **kwargs):
        context = super(HomeView, self).get_context_data(*args, **kwargs)
        context['brand_name'] = 'My Brand'
        return context

    def get(self, request, *args, **kwargs):
        context = self.get_context_data()
        return super(TemplateView, self).render_to_response(context)


class AboutView(TemplateView):
    http_method_names = ['get']
    template_name = "about_page.html"


class LoginView(TemplateView):
    http_method_names = ['get', 'post']
    template_name = 'auth/login.html'

    def get(self, request, *args, **kwargs):
        form = LoginForm()
        context = self.get_context_data()
        context['form'] = form
        context['name'] = 'login'
        context['content'] = ''
        return super(TemplateView, self).render_to_response(context)

    def post(self, request, *args, **kwargs):
        form = LoginForm(request.POST or None)
        context = self.get_context_data()
        context['form'] = form
        print(f'user authenticated:{request.user.is_authenticated()}')
        if form.is_valid():
            print(form.cleaned_data)
            
            username = form.cleaned_data.get('username')
            password = form.cleaned_data.get('password')
            user = authenticate(request, username=username, password=password)

            if user is not None:
                login(request, user)
                # Redirect to success page
                # context['form'] = LoginForm()
                return HttpResponseRedirect('/')
            else:
                # Return an 'invalid login' error message
                print('Error')
            print(f'after authentication. user authenticated:{request.user.is_authenticated()}')
        return super(TemplateView, self).render_to_response(context)

class RegistrationView(TemplateView):
    http_method_names = ['get', 'post']
    template_name = 'auth/register.html'

    def get(self, request, *args, **kwargs):
        form = RegistationForm()
        context = self.get_context_data()
        context['form'] = form
        context['name'] = 'register'
        context['content'] = ''
        return super(TemplateView, self).render_to_response(context)

    def post(self, request, *args, **kwargs):
        form = RegistationForm(request.POST or None)
        if form.is_valid():
            print(form.cleaned_data)
            User = get_user_model()
            username = form.cleaned_data.get('username')
            password = form.cleaned_data.get('password')
            email = form.cleaned_data.get('email')
            new_user = User.objects.create_user(
                username,
                email,
                password
            )
            print(f'new user:{new_user}')
        context = self.get_context_data()
        context['form'] = form
        return super(TemplateView, self).render_to_response(context)


class ContactView(TemplateView):
    http_method_names = ['get', 'post']
    template_name = "contact/view.html"

    def get_context_data(self, *args, **kwargs):
        context = super(ContactView, self).get_context_data(*args, **kwargs)
        return context

    def get(self, request, *args, **kwargs):
        form = ContactForm()
        context = self.get_context_data()
        context['form'] = form
        context['name'] = 'contact'
        context['brand_name'] = 'My Brand'
        return super(TemplateView, self).render_to_response(context)

    def post(self, request, *args, **kwargs):
        # fullname = request.POST.get('fullname',None)
        # email = request.POST.get('email', None)
        # content = request.POST.get('content', None)
        # print(f'fullname: {fullname} email:{email} content:{content}')
        form = ContactForm(request.POST or None)
        if form.is_valid():
            print(form.cleaned_data)
        context = self.get_context_data()
        context['form'] = form
        return super(TemplateView, self).render_to_response(context)
```
Now comes the `urls.py`
```python
"""ecom URL Configuration

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/1.11/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  url(r'^$', views.home, name='home')
Class-based views
    1. Add an import:  from other_app.views import Home
    2. Add a URL to urlpatterns:  url(r'^$', Home.as_view(), name='home')
Including another URLconf
    1. Import the include() function: from django.conf.urls import url, include
    2. Add a URL to urlpatterns:  url(r'^blog/', include('blog.urls'))
"""
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
    url(r'^$', HomeView.as_view()),
    url(r'^about/$', AboutView.as_view()),
    url(r'^contact/$', ContactView.as_view()),
    url(r'^login/$', LoginView.as_view()),
    url(r'^register/$', RegistrationView.as_view()),
    url(r'^products/', include('ecom_product.urls', namespace='ecom_product')),
    url(r'^admin/', admin.site.urls),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```
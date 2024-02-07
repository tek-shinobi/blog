---
title: "Backend Python:Adding Reverse Url Lookup in Django"
date: 2018-09-07T13:39:51+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
Reverse url lookup is very cool feature in Django that allows us to not hard-code urls in templates or controller logic. This helps us change the URLs later in urls.py and not have to make the same changes everywhere in templates and logic. Keep it DRY.

To implement reverse url lookup, you need to do these:

1. Make a namespace (if your urls are inside an app… otherwise omit this step). Making a namespace makes it possible to have multiple apps use the same name for named urls.
1. make the named url

The main url (same level as `settings.py`). Note that the namespace is set here:

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
    url(r'^$', HomeView.as_view()),
    url(r'^about/$', AboutView.as_view()),
    url(r'^contact/$', ContactView.as_view()),
    url(r'^login/$', LoginView.as_view()),
    url(r'^register/$', RegistrationView.as_view()),
    url(r'^products/', include('ecom_product.urls', namespace='ecom_product')),
    url(r'^admin/', admin.site.urls),
] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

```
Then the `urls.py` in the app econ_product:
```python
from django.conf.urls import url

from .views import (
    EcomProductListView,
    # EcomProductDetailView,
    # EcomProductFeaturedListView,
    # EcomProductFeaturedDetailView,
    EcomProductDetailSlugView
)

urlpatterns = [
    # url(r'^featured/(?P<slug>[\w-]+)/$', EcomProductFeaturedDetailView.as_view()),
    # url(r'^featured/(?P<pk>\d+)', EcomProductFeaturedDetailView.as_view()),
    url(r'^(?P<slug>[\w-]+)/$', EcomProductDetailSlugView.as_view(), name='detail'),
    # url(r'^products/(?P<pk>\d+)', EcomProductDetailView.as_view()),
    # url(r'^featured/$', EcomProductFeaturedListView.as_view()),
    url(r'^$', EcomProductListView.as_view(), name='list'),
]
```
Now you can use the url lookup.

Note, the lookup is done differently in template and in controller logic. In controller logic, we will need a `reverse()` call to construct the url for us. Like so:
(Here I am doing a reverse url lookup in models.py in the method get_absolute_url like so: 
`reverse('ecom_product:detail', kwargs={'slug':self.slug}))`

Here`: ecom_product` is the namespace and `detail` is the url name in that namespace.
Note also that kwargs passed corresponds to the parameter `slug` in the url
`url(r'^(?P<slug>[\w-]+)/$', EcomProductDetailSlugView.as_view(), name='detail'),`

```python
from django.db import models
import os
import random
from django.db.models.signals import pre_save
from django.urls import reverse
from ecom_utils.utils import unique_slug_generator


def get_filename_ext(filepath):
    base_name = os.path.basename(filepath)
    name, ext = os.path.splitext(base_name)
    return name, ext


def upload_image_path(instance, filename):
    new_filename = random.randint(1,348753947539457)
    name, ext = get_filename_ext(filename)
    final_filename = f'{new_filename}{ext}'
    return f'ecom_products/{new_filename}/{final_filename}'


class EcomProductQuerySet(models.query.QuerySet):
    def get_by_id(self, id):
        qs = self.filter(id=id)
        if qs.count() == 1:
            return qs.first()
        return None
    
    def featured(self):
        return self.filter(featured=True)

    def active(self):
        return self.filter(active=True)


class EcomProduct(models.Model):
    title = models.CharField(max_length=100)
    slug = models.SlugField(blank=True)
    description = models.TextField(null=False, blank=False)
    price = models.DecimalField(decimal_places=2, max_digits=10, null=False, blank=False, default=1.00)
    image = models.ImageField(upload_to=upload_image_path, null=True, blank=True)
    featured = models.BooleanField(default=False)
    active = models.BooleanField(default=True)

    # objects = EcomProductManager()
    objects = EcomProductQuerySet.as_manager()


    def get_absolute_url(self):
        # return f'/products/{self.slug}/'
        return reverse('ecom_product:detail', kwargs={'slug':self.slug})

    def __str__(self):
        return self.title


def ecom_product_pre_save_receiver(sender, instance, *args, **kwargs):
    if not instance.slug:
        instance.slug = unique_slug_generator(instance)


pre_save.connect(ecom_product_pre_save_receiver, sender=EcomProduct)
```
In a template, we do the url lookup like so 
`{% url 'ecom_product:detail' slug=instance.slug %}:`
```html
<div class="card" style="width: 18rem;">
    {% if instance.image %}
    <a href="{{instance.get_absolute_url}}">
        <img src="{{instance.image.url}}" class="card-img-top" alt="{{instance.title}} logo">
    </a>
    {% endif %}
    <div class="card-body">
      <h5 class="card-title">{{instance.title}}</h5>
      <p class="card-text">Some quick example text to build on the card title and make up the bulk of the card's content.</p>
      <a href="{% url 'ecom_product:detail' slug=instance.slug %}" class="btn btn-primary">View</a>
    </div>
</div>
```
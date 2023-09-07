---
title: "Backend Python:Django Custom Queryset vs Custom Manager for Database Queries"
date: 2018-09-07T13:30:55+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---

Note: __this applies to Django >= 1.7__

From the docs:

>A Manager is the interface through which database query operations are provided to Django models. At least one Manager exists for every model in a Django application.
>
>There are two reasons you might want to customize a Manager: to add extra Manager methods, and/or to modify the initial QuerySet the Manager returns.

Since Django 1.7 and onwards, we no longer need to create custom manager and custom queryset separately. We can turn custom queryset into manager like so.
`objects = EcomProductQuerySet.as_manager()`

Custom Queryset is a way to simplify the database queries. For example if we have a database model like so:
```python
from django.db import models
import os
import random

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
    description = models.TextField(null=False, blank=False)
    price = models.DecimalField(decimal_places=2, max_digits=10, null=False, blank=False, default=1.00)
    image = models.ImageField(upload_to=upload_image_path, null=True, blank=True)
    featured = models.BooleanField(default=False)
    active = models.BooleanField(default=True)

    # objects = EcomProductManager()
    objects = EcomProductQuerySet.as_manager()

    def __str__(self):
        return self.title
```
Here, I could have used `EcomProduct.objects.filter(featured=True)` to get only the featured products. Instead, we created a method in custom queryset called featured. Now we can get the featured products like so:
`EcomProduct.objects.featured()`. This might be a trivial example, but this concept really makes a difference in larger queries.

## Query Chaining
Another excellent feature of a queryset is its ability to chain. An example of it is here. In the listview, we create a queryset that only shows `featured` items AMONGST `active` items. These chains can be arbitrarily long.
```python
class EcomProductFeaturedListView(ListView):
    queryset = EcomProduct.objects.active().featured()
    template_name = 'ecom_product/list.html'
```
Note that we could have made this particular query like so (without any custom querysets), but using these custom querysets makes our queries look more DRY and cleaner.

`EcomProduct.objects.filter(active=True, featured=True)`


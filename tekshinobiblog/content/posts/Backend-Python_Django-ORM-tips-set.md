---
title: "Backend Python:Django ORM Tips: _set meaning in Django ORM relationship"
date: 2018-09-07T11:18:20+03:00
draft: false
categories: ["backend", "python"]
tags: ["python","django"]
---

_set is associated with reverse relation on a model.

Django allows you to access reverse relations on a model. By default, Django creates a manager (`RelatedManager`) on your model to handle this, named `<model>_set`, where `<model>` is your model name in lowercase.

Excellent link on StackOverflow here:
https://stackoverflow.com/questions/25386119/whats-the-difference-between-a-onetoone-manytomany-and-a-foreignkey-field-in-d

If we have these models:
```python
class User(models.Model):
    username = models.CharField(max_length=100, unique=True)
    companies = models.ManyToManyField('Company', blank=True)

class Company(models.Model):
    name = models.CharField(max_length=255)
```
In Django,
>“It doesn’t matter which model has the ManyToManyField, but you should only put it in one of the models — not both.”.

So, to get all the companies associated with a User, we can do:
`User.companies.all()`
But the reverse is a bit tricky. That is, how to get all users associated with a company.

Very easy. Get the reverse relationship using _set
`company.user_set.all()`
will return a `QuerySet` of `User` objects that belong to a particular company. By default you use `modelname_set` to reverse the relationship, but you can override this be providing a `related_name` as a parameter when defining the model, i.e.
```python
class User(models.Model):
    username = models.CharField(max_length=100, unique=True)
    companies = models.ManyToManyField('Company', blank=True, related_name="users")

class Company(models.Model):
    name = models.CharField(max_length=255)
```
Then you can get the reverse without using _set like so:
`company.users.all()`
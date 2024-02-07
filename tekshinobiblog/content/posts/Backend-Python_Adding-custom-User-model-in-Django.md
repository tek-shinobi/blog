---
title: "Backend Python:Adding Custom User Model in Django"
date: 2018-09-07T12:04:04+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
## Part 1
One of the first things I do after creating a new Django or Django Rest Framework project is to create a custom User model. Part 1 deals with case when you add custom User model right at the start of a project. In [ part 2 ](#part-2), we deal with scenario when custom User model is added later in the project.

## What exactly is a User model for?
Django uses a User model for only one purpose, user authentication. If the project is really small and not ever going to scale, you can tack on non-authentication related stuff as well, like user address, user profile data etc but this is not recommended. I like to keep my models simple and they only contain what is necessary for authentication. Everything else, I move it to other models like UserProfile etc

## Why custom user model?
Well, Django’s default created User model makes certain crippling assumptions. Like it will only use username for login. If you already put your project in production with active users and then down the line wanted to change it to use email for authentication, this will not be easy. If the active users are too many, it might even be impossible to change. It is so because it involves deleting the old database models and create new ones that use the new user model. So, we must create a custom user model right at the start and switch over to email for authentication. This also makes it easy to add other customization later on. Do this before doing your first migration.

## Create Custom User Model
Follow these steps.

1. `create a new app`, only for managing custom user model. I call it `accounts` in most of my projects.
1. `create User Model and Model Manager`. In my case, I called the model as `MzkUser` and model manager as `MzkUserManager`. Note, it is important to __NOT__ name them as `User` and `UserManager` as these might become reserved words in future. Since my project was about a music app, I prepended it with `Mzk`. In the code below, all models and model-methods and properties are required
```python
from django.db import models
from django.contrib.auth.models import (
    AbstractBaseUser, BaseUserManager
)

class MzkUserManager(BaseUserManager):
    # create_user takes all REQUIRED_FIELDS and USERNAME_FIELD
    def create_user(self, email, password=None, is_active=True, is_staff=False, is_admin=False):
        if not email:
            raise ValueError("Users must have an email address")
        if not password:
            raise ValueError("Users must have a password")

        user_obj = self.model(
            email = self.normalize_email(email)
        )
        user_obj.set_password(password)
        user_obj.staff = is_staff
        user_obj.admin = is_admin
        user_obj.active = is_active
        user_obj.save(using=self._db)
        return user_obj
    
    def create_staff_user(self, email, password=None):
        user = self.create_user(
            email,
            password=password,
            is_staff=True
        )
        return user

    def create_superuser(self, email, password=None):
        user = self.create_user(
            email,
            password=password,
            is_staff=True,
            is_admin=True
        )
        return user

# password is inherited from AbstractBaseUser
class MzkUser(AbstractBaseUser):
    email = models.EmailField(default='abc@gmail.com', max_length=255, unique=True)
    active = models.BooleanField(default=True)
    staff = models.BooleanField(default=False)
    admin = models.BooleanField(default=False)
    timestamp = models.DateTimeField(auto_now_add=True)
    
    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = []

    objects = MzkUserManager()

    def __str__(self):
        return self.email
    
    def get_full_name(self):
        return self.email

    def get_short_name(self):
        return self.email

    # we don't use object/module level permissions
    def has_perm(self, perm, obj=None):
        return True

    # we don't use module level permissions
    def has_module_perms(self, app_label):
        return True

    @property
    def is_staff(self):
        return self.staff
    
    @property
    def is_admin(self):
        return self.admin

    @property
    def is_active(self):
        return self.active
```
3. `settings.py`: Register the accounts app by adding it to `INSTALLED_APPS`. Then add the following line:
`AUTH_USER_MODEL = 'accounts.MzkUser'`
This will indicate which custom user model we are using
3. Run makemigrations and migrate
3. `accounts/admin.py`: The above steps will enable the project to use custom user model. But if we went to the admin page, this will still not be visible (admin will use this model to add new users). To be able to see this on admin page, we need to also register this model with admin panel like so:
```python
from django.contrib import admin
from django.contrib.auth import get_user_model

User = get_user_model()


class UserAdmin(admin.ModelAdmin):
    class Meta:
        model = User


admin.site.register(User)
```
6. Now run migrations, create superuser and then goto admin page to see the new user model.

Note: Here, I am not using object-level permissions. Hence am returning `True` in `has_module_perms` method. You will anyways want to use a library like `django-guardian` for managing object-level permissions.

You will also want to revisit `has_perm` method when you implement user level permissions. For simplicity, I am returning `True` here.

## Part 2
## Adding Custom User model in Django – (using fixtures)
In this part we are discussing what to do when we add custom user model later in the project.

Note that in this scenario, we are in a very non-ideal situation. To keep unknown surprises at the minimum, we will still delete old database but we will use the concept of fixtures to save as much information as we can from the current database, that can be reused later.

For example, if we have an eCommerce site with a lot of products and their details in the database, we can save this info as this is not really connected to any specific user. Then we can remove the database, let Django create a new database with the new custom user model and then you can bring in the products and their details saved earlier. All this is possible using fixtures.

If you want to dump entire database as json, this is the fixture for it:
`python manage.py dumpdata --format json --indent 4`

The above will dump all the info to the console. You can save all this into a file like so:
`python manage.py dumpdata --format json --indent 4 > all_database.json`

The above line will create an `all_database.json` file at the root of project wih all the database content in it.

I generally like to make my dumps separately for specific tables. Like if I have an app called `ecom_product` and in its `models.py` file, I have a model called `Product`, I will save this specific model like so:
Create a directory called `fixtures` in the app folder `ecom_product`. Then run this command
```shell
python manage.py dumpdata ecom_product.Product --format json --indent 4 > ecom_product/fixtures/ecom_product.json
```
Bingo!, you have dumped all the data from your Products table.

Now, goto [ part-1 ](#part-1). And do steps 1 through 3. Then delete/move the old database so Django can create a new one. Before running step 4, remove all migrations. To do this, goto each app’s migrations folder and remove everything except `__init__.py` file (Also remove `__pycache__` folder inside migrations folder). In reality, you only need to remove the migrations related to the old user model. I remove all migrations just in case.

Now you can go ahead with steps 4 and onwards.

Do all the migrations.
Recreate superuser (this time using email). Goto admin panel to see all looks good.

Then at last we are ready to load the saved dumps back into the new database.

Here is how to load dumps back into the database.
```shell
python manage.py loaddata ecom_product/fixtures/ecom_product.json
```
---
title: "Backend Python:Writing Custom Django Management Commands"
date: 2018-09-07T14:02:37+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---

Any command given with `manage.py` is called management command. Django comes with many built-in management commands like `runserver`, `startapp` etc. To see a full list of built-in management commands, type:
`python manage.py help`

The benefit of management command script is that this script executes within Django environment. You can run all of Django ORM queries within this script. You can import models. You have access to project resources.

In this tutorial we will see how to populate a table with some contents of a csv file.

First lets look at the file-structure:
First create a folder called management. Create a `__init__.py` file inside it. Then create a folder called commands inside management and create a `__init__.py` inside it.
```
├── management
│   ├── commands
│   │   ├── __init__.py
│   │   
│   └── __init__.py
```
__tip__: It is always useful to either put management folder in an existing app or create a new app within your django project just for management commands. Keeping management inside an app (which is registered in `INSTALLED_APPS` in `settings.py`) makes it easy for the command to be located.

In my Django project, I created an app called my_app (`python manage.py startapp my_app`). I have put my management folder inside it. Then I create a file called `test_command.py` inside commands folder. The structure looks like so:

```
my_app
├── admin.py
├── apps.py
├── __init__.py
├── management
│   ├── commands
│   │   ├── __init__.py
│   │   └── test_command.py
│   └── __init__.py
├── migrations
│   ├── 0001_initial.py
│   └── __init__.py
├── models.py
├── tests.py
└── views.py
```
The management command gets the name of the file. So since the filename is `test_command.py`, the management command will be called `test_command`. We will be launching this command like so:
`python manage.py test_command`

Now lets write the most basic custom management command.
```python
from django.core.management.base import BaseCommand


class Command(BaseCommand):
    help = 'populates currencies table'

    def handle(self, *args, **options):
        self.stdout.write(f'hello world')
```
All management commands are subclasses of BaseCommand. The file always has a class called Command which is a subclass of BaseCommand. And this class always has a method
`def handle(self, *args, **options):`

Lets execute the management command.
`python manage.py test_command`

You will see ‘hello world’ printed. Note that inside management commands, we use stdout for printing to console.

## Arguments
We can pass arguments with management commands. There are three kinds of arguments.

1. __Positional arguments__: required. command will not run if these not passed
1. __optional arguments__: optional. command will run even if these not passed
1. __Flag Arguments__ optional boolean

Arguments are parsed using Python’s [ argparse ](https://docs.python.org/3/library/argparse.html) library.
For simple use cases, we will be able to use add_arguments convenience method in Django. For more customized use cases we will need to work with argparse library.

For handling arguments, we need to add a method called `add_arguments`.

## Mandatory Arguments (Also called Positional Arguments)

We will create a management command called `populate_currencies.py`. This command will read a csv file with currency codes and fill a database table called `Currencies`. We will pass a mandatory argument called filename that will be the name of csv file we want to read.

since it is positional, we can use it like so:
`python manage.py populate_currencies currencies.csv`

See below for `currencies.csv` file

First, I’ll create a model to hold the currency codes. Add this to `models.py` file:
```python
class Currencies(models.Model):
    """
    Contains all the valid currency codes
    """
    code = models.CharField(
        max_length=3,
        unique=True,
        null=False,
        blank=False
    )
    name = models.CharField(
        max_length=50,
        null=True,
        blank=True,
        unique=False
    )

    def __str__(self):
        return f'{self.code} {self.name}'
```

Then, I’ll paste a simple csv file here for your reference:
```csv
Country,CountryCode,Currency,Code
New Zealand,NZ,New Zealand Dollars,NZD
Australian,AU,Australian Dollars,AUD
Ireland,IE,Euros,EUR
United Kingdom,GB,Sterling,GBP
Japan,JP,Japanese Yen,JPY
Virgin Islands (US),VI,USD,USD
Hong Kong,HK,HKD,HKD
Canada,CA,Canadian Dollar,CAD
```
Put this csv file in the same folder as the management command `populate_currencies.py`. This is not a requirement but to keep matters simple, thats where we keep the csv file.

OK, so now `populate_currencies.py` for reading this csv and populating `Currencies` table is:
```python
from django.core.management.base import BaseCommand, CommandError
from django.apps import apps
import csv
import os

from currencyrates.models import Currencies


class Command(BaseCommand):
    help = 'populates currencies table'

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.model_name = Currencies

    def add_arguments(self, parser):
        parser.add_argument('filename', type=str, help='filename for csv file')

    def get_current_app_path(self):
        return apps.get_app_config('currencyrates').path

    def get_csv_file(self, filename):
        app_path = self.get_current_app_path()
        file_path = os.path.join(app_path, "management",
                                 "commands", filename)
        return file_path

    def clear_model(self):
        try:
            self.model_name.objects.all().delete()
        except Exception as e:
            raise CommandError(
                f'Error in clearing {self.model_name}: {str(e)}'
            )

    def insert_currency_to_db(self, data):
        try:
            self.model_name.objects.create(
                code=data["code"],
                name=data["name"],
            )
        except Exception as e:
            raise CommandError(
                f'Error in inserting {self.model_name}: {str(e)}'
            )

    def handle(self, *args, **kwargs):
        filename = kwargs['filename']
        self.stdout.write(self.style.SUCCESS(f'filename:{filename}'))
        file_path = self.get_csv_file(filename)
        line_count = 0
        currency_code = []
        try:
            with open(file_path) as csv_file:
                csv_reader = csv.reader(csv_file, delimiter=',')
                self.clear_model()
                for row in csv_reader:
                    if row != '' and line_count >= 1:
                        data = {}
                        data['name'] = row[2]
                        data['code'] = row[3]
                        if data['code'] not in currency_code:
                            currency_code.append(data['code'])
                            self.insert_currency_to_db(data)
                    line_count += 1
            self.stdout.write(
                self.style.SUCCESS(
                    f'{line_count} entries added to Currencies'
                )
            )
        except FileNotFoundError:
            raise CommandError(f'File {file_path} does not exist')
```
In this file, here is where we add the mandatory argument:
```python
def add_arguments(self, parser):
parser.add_argument('filename', type=str, help='filename for csv file')
```

Also note that we can set Styles for console outputs like so:
```python
self.style.SUCCESS(f'{line_count} entries added to Currencies')
```

Please check Epilogue for an entire list of styles.

We launch the command like so:
`python manage.py populate_currencies currencies.csv`

Due to its positional nature, filename argument is set to `currencies.csv`

## Optional Arguments
The optional (and named) arguments can be passed in any order. In the example below you will find the definition of an argument named `prefix`, which will be used to compose the username field:

__management/commands/create_users.py__
```python
from django.contrib.auth.models import User
from django.core.management.base import BaseCommand
from django.utils.crypto import get_random_string

class Command(BaseCommand):
    help = 'Create random users'

    def add_arguments(self, parser):
        parser.add_argument('total', type=int, help='Indicates the number of users to be created')

        # Optional argument
        parser.add_argument('-p', '--prefix', type=str, help='Define a username prefix', )

    def handle(self, *args, **kwargs):
        total = kwargs['total']
        prefix = kwargs['prefix']

        for i in range(total):
            if prefix:
                username = '{prefix}_{random_string}'.format(prefix=prefix, random_string=get_random_string())
            else:
                username = get_random_string()
            User.objects.create_user(username=username, email='', password='123')
```
__Usage__
`python manage.py create_users 10 --prefix custom_user`

or

`python manage.py create_users 10 -p custom_user`

If the prefix is used, the username field will be created as __custom_user_xCVGn3yt56h__. If not prefix, it will be created simply as __xCVGn3yt56h__ – a random string.

## Flag Arguments
Another type of optional arguments are flags, which are used to handle boolean values. Let’s say we want to add an `--admin` flag, to instruct our command to create a super user or to create a regular user if the flag is not present.

__management/commands/create_users.py__
```python
from django.contrib.auth.models import User
from django.core.management.base import BaseCommand
from django.utils.crypto import get_random_string

class Command(BaseCommand):
    help = 'Create random users'

    def add_arguments(self, parser):
        parser.add_argument('total', type=int, help='Indicates the number of users to be created')
        parser.add_argument('-p', '--prefix', type=str, help='Define a username prefix')
        parser.add_argument('-a', '--admin', action='store_true', help='Create an admin account')

    def handle(self, *args, **kwargs):
        total = kwargs['total']
        prefix = kwargs['prefix']
        admin = kwargs['admin']

        for i in range(total):
            if prefix:
                username = '{prefix}_{random_string}'.format(prefix=prefix, random_string=get_random_string())
            else:
                username = get_random_string()

            if admin:
                User.objects.create_superuser(username=username, email='', password='123')
            else:
                User.objects.create_user(username=username, email='', password='123')
```
Note:`action='store_true'` indicates, default value of true. This is straight from argparser (Python library). See [ here ](https://docs.python.org/3/library/argparse.html) for details.

__Usage__:

`python manage.py create_users 2 --admin`
Or
`python manage.py create_users 2 -a`

## Management command automation
Django management commands are typically run from the command line, requiring human intervention. However, there can be times when it’s helpful or necessary to automate the execution of management commands from other locations (e.g. a Django view method or shell).

For example, if a user uploads an image in a Django application and you want the image to become publicly accessible, you’ll need to run the `collectstatic` command so the image makes its way to the public consolidation location (STATIC_ROOT) . Similarly, you may want to run a `cleanuprofile` command every time a user logs in.

To automate the execution of management commands Django offers the `django.core.management.call_command()` method. Below illustrates the various ways in which you can use the `call_command()` method.

__Django management automation with call_command()__
```python
from django.core import management

# Option 1, no arguments
management.call_command('sendtestemails')

# Option 2, no pause to wait for input
management.call_command('collectstatic', interactive=False)

# Option 3, command input with Command()
from django.core.management.commands import loaddata
management.call_command(loaddata.Command(), 'stores', verbosity=0)

# Option 4, positional and named command arguments
management.call_command('cleanupdatastores', 1, delete=True)

```
The first option executes a management command without any arguments. The second option uses the `interactive=False` argument to indicate the command must not pause for user input (e.g. collectstatic always asks if you’re sure if you want to overwrite pre-existing files, the `interactive=False` argument avoids this pause and need for input).

The third option invokes the management command by first importing it and then invoking its Command() class directly vs. using the command string value. And finally, the fourth option — just like the third — in listing 5-35, uses a positional argument — declared as a standalone value (e.g. 'stores', 1) and a named argument — declared as a key=value (e.g. verbosity=0, delete=True).

## Epilogue
A management command to display an entire list of styles is like so:
```python
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = 'Show all available styles'

    def handle(self, *args, **kwargs):
        self.stdout.write(self.style.ERROR('error - A major error.'))
        self.stdout.write(self.style.NOTICE('notice - A minor error.'))
        self.stdout.write(self.style.SUCCESS('success - A success.'))
        self.stdout.write(self.style.WARNING('warning - A warning.'))
        self.stdout.write(self.style.SQL_FIELD('sql_field - The name of a model field in SQL.'))
        self.stdout.write(self.style.SQL_COLTYPE('sql_coltype - The type of a model field in SQL.'))
        self.stdout.write(self.style.SQL_KEYWORD('sql_keyword - An SQL keyword.'))
        self.stdout.write(self.style.SQL_TABLE('sql_table - The name of a model in SQL.'))
        self.stdout.write(self.style.HTTP_INFO('http_info - A 1XX HTTP Informational server response.'))
        self.stdout.write(self.style.HTTP_SUCCESS('http_success - A 2XX HTTP Success server response.'))
        self.stdout.write(self.style.HTTP_NOT_MODIFIED('http_not_modified - A 304 HTTP Not Modified server response.'))
        self.stdout.write(self.style.HTTP_REDIRECT('http_redirect - A 3XX HTTP Redirect server response other than 304.'))
        self.stdout.write(self.style.HTTP_NOT_FOUND('http_not_found - A 404 HTTP Not Found server response.'))
        self.stdout.write(self.style.HTTP_BAD_REQUEST('http_bad_request - A 4XX HTTP Bad Request server response other than 404.'))
        self.stdout.write(self.style.HTTP_SERVER_ERROR('http_server_error - A 5XX HTTP Server Error response.'))
        self.stdout.write(self.style.MIGRATE_HEADING('migrate_heading - A heading in a migrations management command.'))
        self.stdout.write(self.style.MIGRATE_LABEL('migrate_label - A migration name.'))
```
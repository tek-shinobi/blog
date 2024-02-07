---
title: "Backend Python:Using Pytest With Django and Django Rest Framework"
date: 2018-09-07T19:23:46+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python"]
---

## What is Pytest
Pytest is testing framework for Python. Very popular with Django.

### Killer feature : Fixtures
Fixtures are the killer feature of Pytest. Fixtures are functions that run before and after each test, like setUp and tearDown in unitest and labelled pytest killer feature. Fixtures are used for data configuration, connection/disconnection of databases, calling extra actions, and so on.

All fixtures have scope argument with available values:
- function run once per test
- class run once per class of tests
- module run once per module
- session run once per session

__Note__: Default value of scope is __function__

### How to create fixtures
Example of simple fixture creation:
```python
import pytest


@pytest.fixture
def function_fixture():
   print('Fixture for each test')
   return 1


@pytest.fixture(scope='module')
def module_fixture():
   print('Fixture for module')
   return 2
```
Another kind of fixture is yield fixture which provides access to test before and after the run, analogous to setUp and tearDown.

Example of simple yield fixture creation:
```python
import pytest

@pytest.fixture
def simple_yield_fixture():
   print('setUp part')
   yield 3
   print('tearDown part')
```

Note: normal fixtures can use yield directly so the yield_fixture decorator is no longer needed and considered deprecated.

### conftest.py
conftest.py is the file present in the root of tests folder. Here put all the fixtures that you want to be present in all of you test modules.

Example of fixture creation in conftest.py:

```python
import os
import django
import pytest


def pytest_configure():
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'tests.settings')
    django.setup()


@pytest.fixture
def json_error():
    data = {'success': False, 'error': {'info': 'Invalid json'}}
    return data


@pytest.fixture
def json_success():
    data = {
        'success': True, 'rates': {
            'AT': {
                'country_name': 'Austria', 'standard_rate': 20,
                'reduced_rates': {'foodstuffs': 10, 'books': 10}},
            "BE": {
                "country_name": "Belgium",
                "standard_rate": 21,
                "reduced_rates": {
                    "restaurants": 12,
                    "foodstuffs": 6,
                    "books": 6,
                    "water": 6,
                    "pharmaceuticals": 6,
                    "medical": 6,
                    "newspapers": 6,
                    "hotels": 6,
                    "admission to cultural events": 6,
                    "admission to entertainment events": 6
                }
            },
        }}
    return data


@pytest.fixture
def json_success_without_reduced_rates():
    data = {
        'success': True, 'rates': {
            'AZ': {
                'country_name': 'Austria', 'standard_rate': 20,
                'reduced_rates': None}}}
    return data


@pytest.fixture
def json_types_success():
    data = {'success': True, 'types': ['books', 'wine', 'medicine']}
    return data
```

Here, I created fixtures like json_error, json_success etc. If I now create test modules like test_tekvat.py, test_tekmoney.py etc, these fixtures will be globally available in all of these modules.

## How to use Fixtures with test in Pytest?
To use fixture in test, you can put fixture name as function argument:
```python
import pytest

@pytest.fixture
def function_fixture():
   print('Fixture for each test')
   return 1

@pytest.fixture
def simple_yield_fixture():
   print('setUp part')
   yield 3
   print('tearDown part')

def test_function_fixture(function_fixture):
  assert function_fixture == 1

def test_yield_fixture(simple_yield_fixture):
  assert simple_yield_fixture == 3
```

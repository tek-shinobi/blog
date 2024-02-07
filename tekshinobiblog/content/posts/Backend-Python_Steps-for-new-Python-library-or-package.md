---
title: "Backend Python:Steps for New Python Library or Package"
date: 2018-09-07T10:49:17+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python"]
---

Here are the steps.

Create repo on github with license, gitignore and readme stubs.

Clone the repo in the work area.

Lets say, my new library is called `new-library`

`new-library` is the repo name.

Inside the cloned repo, create virtual env by running:
`pipenv install --python 3.8`

Above step creates a virtualenv using python 3.8 (already installed on my system)

now install all the required packages. The basic ones I always install are:
pytest, pytest-cov, typing, codecov, mypy and flake8

```shell
pipenv shell
pipenv install pytest pytest-cov typing codecov mypy flake8
```
Add
```
# IDE dirs
.idea/
.vscode/
```
to `.gitignore`

create a directory called new-library (inside the directory new-library .. this is also the repo). Inside this, create a file called `__init__.py`. new-library is now a Python package. I tend to have my `__init__.py` blank.

lets create a file called `currency.py` inside this directory so that we have something to work with.

```python
from typing import Union
from decimal import Decimal
import warnings

Dint = Union[Decimal, int]


class Currency:
    """Handles money per specified currency"""

    __slots__ = ('amount', 'currency')

    def __init__(self, amount: Dint, currency: str) -> None:
        if isinstance(amount, float):
            warnings.warn(
                SyntaxWarning(  # pragma: no cover
                    'float value detected. Please use Decimal instead.'
                ),
                stacklevel=2
            )
        self.amount = amount
        self.currency = currency.upper() or 'USD'

    def __str__(self) -> str:
        return f'{str(self.amount)} {self.currency}'
```
Now lets create a tests directory right under the repo directory (called new-library). This tests directory is at the same level as .gitignore and README.md files.

Inside this directory, create `__init__.py`. Then create a file called `test_currency.py`

```python
import pytest

from new-library.currency import Currency


def test_str():
    currency = Currency(5, 'USD')
    assert str(currency) == '5 USD'
```
Now create a `.coveragerc` file to configure code coverage. This file is also at the root level (same level as `.gitignore`)
```
[run]
branch = 1
omit = */test*.py
source = new-library

[report]
exclude_lines =
    pragma: no cover
    raise NotImplementedError
    return NotImplemented

```
Now create a setup.cfg file where you can put configuration details. Again at the root.

```
[metadata]
description-file = README.md

[flake8]
exclude = .git,*migrations*,setup.py
max-line-length = 79
```
Now create a setup.py file. Again at root level.
```python
from distutils.core import setup
setup(
    name='new-library',         # How you named your package folder (MyLib)
    packages=['new-library'],   # Chose the same as "name"
    version='1.0',      # Start with a small number and increase it with every change you make
    license='MIT',        # Chose a license from here: https://help.github.com/articles/licensing-a-repository
    description='Handles money',   # Give a short description about your library
    author='tek shinobi',                   # Type in your name
    author_email='hello@gmail.com',      # Type in your E-Mail
    url='https://github.com/tek-shinobi/new-library',   # Provide either the link to your github or to your website
    keywords=['money', 'currency'],   # Keywords that define your package best
    install_requires=[            # I get to this in a second
        'babel>=2.5.0',
        'typing>=3.6.0',
    ],
    platforms=['any'],
    classifiers=[
        'Development Status :: 3 - Alpha',      # Chose either "3 - Alpha", "4 - Beta" or "5 - Production/Stable" as the current state of your package
        'Intended Audience :: Developers',      # Define that your audience are developers
        'Topic :: Software Development :: Libraries :: Python Modules',
        'License :: OSI Approved :: MIT License',   # Again, pick a license
        'Programming Language :: Python :: 3',      # Specify which python versions that you want to support
        'Programming Language :: Python :: 3.6',
        'Programming Language :: Python :: 3.8',
        'Operating System :: OS Independent',
    ]
)
```
Now, lastly create `.travis.yml` file. Again at root level.
```yaml
language: python
python:
  - 3.6
  - 3.8
install:
  - pip install pipenv
  - pipenv install
  - python setup.py install
  - pip install mypy pytest pytest-cov flake8
  - pip install codecov
script:
  - flake8 new-library
  - pytest --cov
  - mypy new-library --ignore-missing-imports
after_success:
  - codecov
```

Don’t forget to register and enable the repo in Travis CI dashboard before checking in the project. Now, after you check in all changes to github, you will see that build gets triggered in travis.

locally, you can run the tests with coverage using:
`pytest --cov=new-library tests/`

You can check the compile with type safety:
`mypy new-library --ignore-missing-imports`

You can check linting:
`flake8 new-library`

## Steps for PyPi package
first install check-manifest for creating MANIFEST.in
`pipenv install --dev check-manifest`

create a source distribution for PyPi
`python setup.py sdist`

now check the distribution created for all the files:
`tar tzf dist/read-only-attributes-1.0.tar.gz`

It will be missing Pipfile, Pipfile.lock and LICENSE files. You need to create a MANIFEST.in to include these files.
`check-manifest --create`

Now re-create source distribution to include the files listed in the MANIFEST.in and then check to see that the are in the dist
`python setup.py sdist`
`tar tzf dist/read-only-attributes-1.0.tar.gz`

Now we are ready to publish it.
So lets build the .whl distribution file for PyPi
`python setup.py bdist_wheel sdist`

check the dist folder to ensure that .gz and .whl files were created
`ls dist/`

Now we are ready to upload to PyPi.
Since we need twine to upload to PyPi, install it first
`pipenv install --dev twine`

now upload it:
`twine upload dist/*`
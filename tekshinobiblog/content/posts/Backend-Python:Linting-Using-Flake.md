---
title: "Backend Python:Linting Using Flake"
date: 2018-09-07T10:08:20+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python"]
---
Linting is a must for any python project.

Two options are used most (in order of usage and popularity)
1. flake8 – most used by open source python projects
1. pylint – enabled by default in many IDEs like Visual Studio Code

Many folks run both.

I chose flake8. Primary reason is that out of the box, pylint barks at everything in my code. The signal to noise ratio is god-awful. It needs to be configured via .pylintrc before you can actually use it.

flake8 on the other hand only raises sensible issues by default. Rest you can configure.

Both implement PEP8 style guide.

Installation (works on all OSes):
1. First activate your virtual environment.
1. Then use `pip install flake8`
1. Then in Visual Studio Code, goto File->Preferences->Settings : type lint in search box. Choose Python. Search for flake8. Check Python>Linting:Flake8Enabled


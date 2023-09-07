---
title: "Backend Python:Django Sessions for Anonymous Users"
date: 2018-09-07T11:39:04+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
We know that all logged in users are connected to a session. This is something done for you by Django middlewares.

Similarly, for anonymous users (not logged in), every time the server receives a request, django creates a session object (meaning, an object with session_key, session_data and expire_data values). But the catch here is that you won’t see this session object in the django_session table because django sessions are [ saved only when modified ](https://docs.djangoproject.com/en/1.11/topics/http/sessions/#when-sessions-are-saved).

You can choose to save session every request by setting:

`SESSION_SAVE_EVERY_REQUEST = True`

Mode details here: https://docs.djangoproject.com/en/2.1/topics/http/sessions/#when-sessions-are-saved
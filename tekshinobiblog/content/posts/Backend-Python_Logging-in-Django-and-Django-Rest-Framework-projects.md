---
title: "Backend Python:Logging in Django and Django Rest Framework Projects"
date: 2018-09-07T16:02:36+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
Logging is extremely necessary, during development and production. There are two ways to implement logging.

1. In apps with high scale, we log to console and then have some in-the-cloud solution like Sentry to log the messages from the console. This is great because large scale apps often run on multiple servers and we can converge all the logs to one place.
1. Alternately, we can log the messages to file on the server itself. This is usually recommended for smaller scale apps. This is the topic being explored in this post.

Some overview first. Django uses the Python builtin `logging` module. The module needs to be configured first before it can be used.

To be able to have a consistent logging experience across the app, the best place to configure logger is `settings.py`.

So in `settings.py`, import the module like so `import logging`

then configure it like so:
```python
# configure logging
logging.config.dictConfig({
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'console': {
            'format': '%(name)-12s %(levelname)-8s %(message)s'
        },
        'file': {
            'format': '%(asctime)s %(name)-12s %(levelname)-8s %(message)s'
        }
    },
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'console'
        },
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'formatter': 'file',
            'filename': '../currencyconverter/logs/debug.log'
        }
    },
    'loggers': {
        '': {
            'level': 'DEBUG',
            'handlers': ['console', 'file']
        }
    }
})
```
`logging.config.dictConfig` is always set to version 1. Then it has three important parts:

1. handlers: these are like buckets for log messages. They all tap into the same source of incoming messages. One handler can be a file. Another handler can be a console. The same log message is broadcast to all handlers. The hadlers can have filters on them to decide to drop some. By default, they accept all broadcasted messages that match the configured log level. Handlers are wired to formatters.
1. formatters: these format the log messages. Each handler has a corresponding formatter that formats the log message for that handler
1. loggers: this determines the minimum logging level for all the handlers. I always keep it at DEBUG as that is the lowest level
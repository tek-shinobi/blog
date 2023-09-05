---
title: "Python Scrapy:Removing Non Ascii Characters From Text in Python"
date: 2018-09-05T11:13:58+03:00
draft: false 
categories: ["python"]
tags: ["python", "scrapy"]
---

I was handling some text scraped using Scrapy and the text had non-ascii unicode charcters like `\u003e`.
If I did this, it didn't work:
```python
        html_text = response.text.encode('ascii', errors='ignore').decode()
```


Here response.text is the string that contains unicode text (scrapy returns strings encoded in unicode).
The html_text still had non ascii unicode characters like `\u003e`
This worked:

```python
        html_text = response.text.encode('ascii', errors='ignore').decode('unicode-escape')
```

Note that `unicode-escape` part in decode. That made the difference in getting rid of characters like `\u003e` and replacing them with space.
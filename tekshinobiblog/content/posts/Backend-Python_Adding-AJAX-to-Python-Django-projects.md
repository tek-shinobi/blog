---
title: "Backend Python:Adding AJAX to Python Django Projects"
date: 2018-09-07T13:55:19+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python","django"]
---
A very nice explanation on SO regarding what and why about AJAX from Django perspective:
https://stackoverflow.com/questions/20306981/how-do-i-integrate-ajax-with-django-applications

So, specifically in Django, when we use AJAX to do a form POST request, we need a CSRF token. This is a security feature. If we are using Django templates, the templates do the job of generating a csrf token and tacking it along with the POST request data sent to the server. But when we are doing form POST via AJAX call, there are no Django templates involved. Hence this CSRF token generation and insertion has to be done manually. Since this is such a common scenario, Django itself has published javascript methods to do this CSRF token generation and have them added to jQuery.

## Django documentation for AJAX integration:
https://docs.djangoproject.com/en/3.0/ref/csrf/

## Code Snippet
So, from the above link from official Django documentation, below is the code regarding how to do it. Just have this code in the static javascript files area and then include this js file in your `base.html`. This will take care to generate csrf token and add it to all AJAX form POST, so long as they are done via jQuery. In fact, you can use jQuery just to do AJAX in other projects like React and Angular as well.

```javascript
$(document).ready(function(){
    function getCookie(name) {
        var cookieValue = null;
        if (document.cookie && document.cookie !== '') {
            var cookies = document.cookie.split(';');
            for (var i = 0; i < cookies.length; i++) {
                var cookie = cookies[i].trim();
                // Does this cookie string begin with the name we want?
                if (cookie.substring(0, name.length + 1) === (name + '=')) {
                    cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                    break;
                }
            }
        }
        return cookieValue;
    }
    var csrftoken = getCookie('csrftoken');

    function csrfSafeMethod(method) {
        // these HTTP methods do not require CSRF protection
        return (/^(GET|HEAD|OPTIONS|TRACE)$/.test(method));
    }
    $.ajaxSetup({
        beforeSend: function(xhr, settings) {
            if (!csrfSafeMethod(settings.type) && !this.crossDomain) {
                xhr.setRequestHeader("X-CSRFToken", csrftoken);
            }
        }
    });
})
```
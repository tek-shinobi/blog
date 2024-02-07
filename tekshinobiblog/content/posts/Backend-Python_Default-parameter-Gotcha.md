---
title: "Backend Python:Default Parameter Gotcha"
date: 2018-09-07T10:12:33+03:00
draft: false 
categories: ["backend", "python"]
tags: ["python"]
---

Take a look at this code. All it does is return a list with appended parameter. In case no list is supplied, it defaults to an empty list and appends to it. Or is it???
```python
def add_to_list(item, item_list=[]):
    item_list.append(item)
    return item_list

add_to_list("gold") # expected: ["gold"] #actual: ["gold"]
add_to_list("silver") # expected: ["silver"] #actual: ["gold","silver"]
```
Look at the second result. We expected `["silver"]` expecting the item to be appended to an empty list as we never passed in a list and expected a default empty list as in function definition.

But here is the gotcha. Python’s default arguments are evaluated once when the function is defined, not each time the function is called. This means that if you use a mutable default argument and mutate it, you will and have mutated that object for all future calls to the function as well.

## Correct Way
Use sentinel objects like None. These are non-mutable. Then incorporate guards to check for sentinels.
```python
def add_to_list(item, item_list=None):
    if item_list is None:
        item_list = []
    item_list.append(item)
    return item_list

add_to_list("gold") # expected: ["gold"] #actual: ["gold"]
add_to_list("silver") # expected: ["silver"] #actual: ["silver"]
```
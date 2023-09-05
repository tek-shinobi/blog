---
title: "Frontend CSS:When to Use Float vs Position vs Display for Laying Out Elements and Aligning Blocks on Page"
date: 2016-09-05T11:32:27+03:00
draft: false 
categories: ["frontend"]
tags: ["css"]
---

Here are the recommendations:

## Float:

- Use this for content that flows around (i.e. variable and flexible content). A good example is an image surrounded by texts of different lengths. So even if the amount of text can change, but using float will allow it to flow around the image.
- Float is also very useful in laying out major parts of layout like header, footer or sidebar.

## Display:

- Can be useful for aligning page components but you need to account for the extra space around the element(`display: inline-block` will put a default space between elements having it)
- Very useful in creating centered horizontal navs because of the fact that the property `text-align: center` works on aligning child elements as well.
- You also don’t need to worry about changing the stacking order or natural flow of elements on the page because it does not affect the layout flow of elements on the page.

## Position:

- Used when an element needs to be positioned relative to another element.
- Also useful in positioning elements that exist outside the natural document flow like a fixed nav bar.
- Positioning elements to a specific place or spot on  the page.
- DON’T use positioning for page layout since it takes the element out of the stacking order with the exception of relative positioning.

## Note:

- Float, display and position cannot be used together on the same element. You need to choose one or the other option.
- If using position (if it is set to absolute or fixed), then float is ignored.
- If using float, then display is ignored.
---
title: "Frontend CSS:Flexbox in a Nutshell"
date: 2019-09-05T10:22:21+03:00
draft: false 
categories: ["frontend"]
tags: ["css"]
---
Flexbox is aimed at a container based layout where we are primarily focused on arranging container items in a single dimension with some abilities of aligning along the cross-axis as well. Because of this, flexbox is often the preferred tool for laying out items along a single dimension. This is different from CSS-Grid which is a grid based layout (two dimensional) out of the box.

Also note that before flexbox, this alignment was achieved using either inline-block or float or some combination of both. Both methods had their shortcomings.

## Floats as layout tool
Secondly, when specifying the width of floated elements in percentages, their computed widths don’t get rounded consistently across browsers (some browsers round up, some down). This means that, sometimes, sections will drop down below others when not intended, and at other times they can leave an irritating gap at one side.

## inline-block as layout tool
The limitations include an automatic margin around inline-block elements. There are hacks to remove it but its inconvenient. Main limitation is that vertical alignment is difficult to achieve. Using inline-block, there is also no way of having two sibling elements where one has a fixed width and another fluidly fills the remaining space.

## `display:table` and `display:table-cell` as layout tool
Don’t confuse `display: table` and `display: table-cell` with the equivalent HTML elements. These CSS properties merely mimic the layout of their HTML-based brethren. They in no way affect the structure of the HTML.

In years gone by, I’ve found enormous utility in using CSS tables for layout. For one, using `display: table` with a `display: table-cell` child enabled consistent and robust vertical centering of elements. Also, table-cell elements inside table elements space themselves perfectly; they don’t suffer rounding issues like floated elements. You also get browser support all the way back to Internet Explorer 7!

However, there are limitations. Generally, it’s necessary to wrap an extra element around items—to get perfect vertical centering, a table-cell must live inside an element set as a table. It’s also not possible to wrap items set as `display: table-cell` on to multiple lines.

Flexbox is intended to address these limitations.

- It can easily vertically center contents.
- It can change the visual order of elements.
- It can automatically space and align elements within a box, automatically assigning available space between them.
- Items can be laid out in a row, a row going in the reverse direction, a column down the page, or a column going in reverse order up a page.

## Flexbox in production
One of the downsides in using flexbox is that browser support across mobile and desktop browsers is inconsistent. To address this, you will need an auto-prefixer. There are many auto-prefixing solutions available, one that I tend to use is called Autoprefixer (https://github.com/postcss/autoprefixer)

## Container and its items
Lets start with this code fragment:
```html
<body>
 <div>Item 1</div>
 <div>Item 2</div>
 <div>Item 3</div>
</body>
```
Since the flexbox is a container based layout tool, the first requirement is to encapsulate the content into a container. I also added some classes on the container items so they can be specifically targeted.
```html
<body>
 <div class="container">
  <div class="item-1">Item 1</div>
  <div class="item-2">Item 2</div>
  <div class="item-3">Item 3</div>
 </div>
</body>
```
## Container level flexbox commands
We turn flexbox layout on by using this line `display: flex` on the container
```css
.container {
 display: flex;
}
```
Another point to keep in mind is that once a container is marked as flex, all of the direct children of the container automatically become flex-items and the container itself becomes a flex-container. This distinction is important. If you have nested structural hierarchy in HTML, with flex-items themselves being containers having their own items, then those nested containers will also need to be set with `display:flex` so as to enable flexbox layout for that nested container’s items. This makes a bit weird combination where HTML content can be a flex-container and flex-item at the same time, depending on the hierarchy one is at. However if the hierarchy goes really deep, you might want to use a 2-D layout tool like CSS-Grid in conjunction with 1-D tool like flexbox to manage layout of upper hierarchies.

Other interesting container level commands are:

- justify-content : for controlling layout along the main axis (default flex-start)
- align-items: for controlling layout along the cross axis (default stretch)
- flex-direction: for setting the main axis as horizontal (row) or vertical (column) (default row)
- flex-wrap: for turning wrapping on. default is nowrap. Hence if the items can’t fit in the container, they will not to next line but will overflow the bounds of the container, also simply called overflow.

## Flex item commands
Lets start with a bit confusing one: `align-self`. This one is almost same as container level command `align-items`, the only difference being that this command only acts on the item its set where as the container level command works on all the container items. Since CSS has specificity, we can use align-self to have item specific override of container level alignment.

Then we have the trio: `flex-grow`, `flex-shrink`, `flex-basis`. `flex-basis` sets the default width of the item. If the item has enough width that it can stretch beyond default width, the stretching behavior is controlled by flex-grow. If the container does not enough width to display items at default width and some shrinking needs to be done on the items, the shrinking behavior is controlled by flex-shrink. The breakpoint where flex-grow stops and flex-shrink begins is the default width of the item and is controlled by flex-basis. `flex-basis:0`; tells that item not to grow or shrink at all. It is also the default setting if no other flex-basis is declared. `flex-basis:auto`; sets the default width of items based on their contents. Besides these, `flex-basis` can also take percentage based width as well as fixed widths based in pixels. `flex-grow` and `flex-shrink` take 0 or positive float values. Negative values are invalid. Default is 0 for both.

flex-grow and flex-shrink both work on the main axis only, where the main axis within a container is determined by the flex-direction setting. `flex-grow:0`; or `flex-shrink:0`; simply means that the item won’t be resized (stretch or shrink along main-axis) during item size calculation to accommodate flex container’s full main axis size.
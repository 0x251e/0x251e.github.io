---
layout: post
title: "Syntax Highlighting and Tables Test"
date: 2023-12-12 10:00:00 +0000
categories: test
---

This post is a test of syntax highlighting and table rendering in Markdown.

## Code Blocks

### HTML

```html
<!DOCTYPE html>
<html>
<head>
  <title>Test HTML</title>
</head>
<body>
  <h1>Hello, world!</h1>
  <p>This is a test HTML file.</p>
</body>
</html>
```
![Description of the image](../image.JPG)


<p align="center">
  <img src="../image.JPG" alt="Profile Image" width="150"/>
</p>

### Key Elements in the Markdown File

1. **Front Matter**:
   - Specifies layout, title, date, and categories. Adjust these values as necessary.

2. **Code Blocks**:
   - Examples from various languages including HTML, CSS, JavaScript, Python, Ruby, Java, and C++.
   - Each code block is enclosed in triple backticks (\```) with the language specified for syntax highlighting.

3. **Tables**:
   - Two tables: one with a simple format and another with aligned text.
   - The first table demonstrates basic column headers and rows.
   - The second table shows alignment for left, center, and right columns.

### Instructions for Use

1. **Save the File**:
   - Place this file in your `_posts` directory. The file name format should be `YYYY-MM-DD-title.md`.

2. **Build and Preview**:
   - Use `jekyll serve` to preview the post on your local server. Navigate to the URL corresponding to this post to view the rendered content.

3. **Customization**:
   - Modify the content or add more examples as needed to test other features or programming languages.

This Markdown file will help you verify that your syntax highlighting is working correctly and that tables are displayed properly on your Jekyll site.


# First level header

### Third level header    ###

## Second level header ######

Hello        {#id}
-----

# Hello      {#id}

# Hello #    {#id}

> This is a blockquote.
>     on multiple lines
that may be lazy.
>
> This is the second paragraph.

> This is a paragraph.
>
> > A nested blockquote.
>
> ## Headers work
>
> * lists too
>
> and all other block-level elements


> A code block:
>
>     ruby -e 'puts :works'

* list 1 item 1
 * list 1 item 2 (indent 1 space)
  * list 1 item 3 (indent 2 spaces)
   * list 1 item 4  (indent 3 spaces)
    * lazy text belonging to above item 4
1. list 1 item 1
 2. list 1 item 2 (indent 1 space)
  3. list 1 item 3 (indent 2 spaces)
   4. list 1 item 4  (indent 3 spaces)
    5. lazy text belonging to above item 4
* list 1 item 1
  * nested list item 1
  * nested list item 2
* list 1 item 2
  * nested list item 1
1. list 1 item 1
   1. nested list item 1
   2. nested list item 2
10. list 1 item 2
    1. nested list item 1
1. text for this list item
   further text (indent 3 spaces)

10. text for this list item
    further text (indent 4 spaces)


definition term 1
definition term 2
: This is the first line. Since the first non-space characters appears in
column 3, all other lines have to be indented 2 spaces (or lazy syntax may
  be used after an indented line). This tells kramdown that the lines
  belong to the definition.
:       This is the another definition for the same term. It uses a
        different number of spaces for indentation which is okay but
        should generally be avoided.
   : The definition marker is indented 3 spaces which is allowed but
     should also be avoided.

| First cell|Second cell|Third cell
| First | Second | Third |

First | Second | | Fourth |


|-----------------+------------+-----------------+----------------|
| Default aligned |Left aligned| Center aligned  | Right aligned  |
|-----------------|:-----------|:---------------:|---------------:|
| First body part |Second cell | Third cell      | fourth cell    |
| Second line     |foo         | **strong**      | baz            |
| Third line      |quux        | baz             | bar            |
|-----------------+------------+-----------------+----------------|
| Second body     |            |                 |                |
| 2 line          |            |                 |                |
|=================+============+=================+================|
| Footer row      |            |                 |                |
|-----------------+------------+-----------------+----------------|



|---
| Default aligned | Left aligned | Center aligned | Right aligned
|-|:-|:-:|-:
| First body part | Second cell | Third cell | fourth cell
| Second line |foo | **strong** | baz
| Third line |quux | baz | bar
|---
| Second body
| 2 line
|===
| Footer row

*some text*
_some text_
**some text**
__some text__

This is a ***text with light and strong emphasis***.
This **is _emphasized_ as well**.
This *does _not_ work*.
This **does __not__ work either**.



<p align="center">
  <img src="../image.JPG" alt="Profile Image" width="150"/>
</p>



Here comes a ![smiley](../images/smiley.png)! And here
![too](../images/other.png 'Title text'). Or ![here].
With empty alt text ![](../images/see.jpg)

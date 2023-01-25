---
layout: post
title: "Example Post"
date: 2023-01-24
categories: jekyll blogging
featured_image: IMG_1143.png
---

The goal of this article is to add some extra info
about blog writing with _Jekyll_.

## Structure your posts

Use level 2 (`##`) and if necessary level 3 (`###`) titles
to structure your posts.

## Display code snippets

You can display a block of code like the following using triple backticks.
You can also specify the language after the first triple backticks.

```python
def hello(name):
    return f'hello {name}'
```

## Add images

Create an `assets` folder where you can put all your images,
then display them with a link starting with an exclamative mark like this:
`![my inspiring image]({{ "/assets/IMG_1143.png" | relative_url }})`.

![my inspiring image]({{ "/assets/IMG_1143.png" | relative_url }})
_Photo by [Matt Koller](https://mrkoller.github.io)_
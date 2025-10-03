---
title       : "Creating posts, understanding Zola"
description : Understanding Zola static site generator
date        : 2024-03-29
template    : post.html
---

Its been a day since I started working on my static blog with Zola. And frankly speaking, its been really simple. I've been following their onboarding guide from their website and here's what my structure looks like as of now.

```shell
website-root/
    templates/
    content/
    static/
    themes/
    config.toml
```

This directory structure gets created when we run `zola init <WEBSITE_ROOT>`

There's the `templates` directory which is meant for HTML templates for the pages of the site. I have `base.html` which, as the name suggests, is the base template from which all other template would extend. The first one of which is the `index.html` file, which forms the main or landing page for my blog. And there's the `post.html` file which forms the template for the posts that I'm going to write on my blog (like this one). Then there also is `posts-page.html` which would contain the list of links to individual posts.

The `content` directory would contain my actual data files. As of now, I've created a subdir inside it calling `posts`, which will contain all my blog posts. This subdirectory structure forms the basis of the web url / path. For example, the file for this post is in `<WEBSITE_ROOT>/content/posts/creating-posts-understanding-zola.md` which is why its address takes on the structure `<WEBSITE_DOMAIN>/posts/creating-posts-understanding-zola`.

The other directories `themes` and `static` are currently empty as I've not reached the stage where I start applying themes and images to my site.

I've followed the guide for now and haven't changed much in my process. The content of the template files are, for example, pretty much the same as what is present in their guide. I've however changed the subdir `content/posts` from their guided `content/blog` for no apparent reason, other than my own instinct.

I am, however, able to create posts like this one. Yeah its that simple. There is no design, no style; but its exactly what is is. A blog. Simple enough. And simple works for me now.

Until next time,
Cheers!

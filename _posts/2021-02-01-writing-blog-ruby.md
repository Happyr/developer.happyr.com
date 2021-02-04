---
title: Writing a blog in Ruby is a mistake
author: Tobias Nyholm
date: '2021-02-01 15:29:00 +0200'
header:
  image: images/posts/ruby.jpg
  teaser: images/posts/ruby.jpg
---

I've been blogging now and then for over a decade now. I have used Drupal, Wordpress,
plain HTML files, static generators as Jekyll and Sculpin. I've also used Github
pages with and without Readthedocs, Couscous etc. I've used formats from plain text
to markdown to Sphinx. This blog is at the time of writing created with Jekyll and
I hate it.

To be precise, I don't hate Ruby or Jekyll. I hate the fact that I am lost in the
Ruby ecosystem. When I have not written any blog posts in a while and I finally
decide to write something and actually finish the post, then I want to preview my
post (naturally). I run `jekyll serve --watch` and get some errors. I try to install
the blog's dependencies again with `bundle install`. I get some other errors… After
30 minutes of me googling I find myself trying to reinstall OpenSsl…

The problem is obviously that I have not updated the dependencies in years and I
get incompatible versions somehow. An experienced Ruby developer would probably
fix the problem within minutes. That is exactly my point and exactly what I want
to do. I want to be able to face any problem and fix it within minutes. That is
why I now have decided to always stick to an ecosystem that I know.

No more Ruby blogs for me.


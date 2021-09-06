---
title: Performance test, Bref FPM vs Bref + Runtime component
author: Tobias Nyholm
date: '2021-09-06 06:44:00 +0200'
header:
  image: images/posts/rollerblades.jpg
  teaser: images/posts/rollerblades.jpg
  caption: "Photo credit: [**Indira Tjokorda**](https://unsplash.com/@indiratjokorda)"
categories:
- Symfony
- Runtime
- Bref
---

The Symfony Runtime component is AWESOME. I did a talk about it at
[Symfony World 2021](https://live.symfony.com/2021-world/) where I explain how and
why it works. I spent a lot of time on the content and the recording, one can still
watch the replay.

Since that talk, I've been deploying a few applications on [Bref](https://bref.sh/)
using [runtime/bref](https://github.com/php-runtime/bref). I really like building
and deploying applications this way.

I did a small test if there is any performance difference between using Bref's FPM
layer compared to using Bref with the runtime component. I tested with a small
symfony app. The app is using Twig and the security component. The data was collected from
AWS CloudWatch while loading the startpage, login page, some authenticated pages etc.

| Layer        | Type       | Count | Average | Min     | Max      |
|--------------|------------|-------|---------|---------|----------|
| Bref FPM     | Request    |    50 | 4.70 ms | 3.34 ms | 14.80 ms |
| Bref/runtime | Request    |    44 | 2.48 ms | 1.55 ms | 9.94 ms  |
| Bref FPM     | Cold start |     5 | 514 ms  | 484 ms  | 582 ms   |
| Bref/runtime | Cold start |     5 | 535 ms  | 464 ms  | 623 ms   |

From the table above one can see that normal HTTP request (warm Lambda) goes from
4.70 ms to 2.48 ms if one use the Runtime component. The cold starts are a bit more
difficult to collect data from. But with 5 cold starts on each function we can see
that they are pretty much the same.

So what does these results really say? I do not expect that all application will
run 50% faster with the Runtime component. But it is safe to assume that most
application will do at least 2-3 ms better.

There is however one major difference. The Bref FPM uses PHP-FPM to manage the
PHP threads and make sure that everything is clean when a new HTTP requests comes
in. The Bref/runtime is keeping the application loaded between requests. This is
great for performance, but it also means that you application may share memory between
requests. Ie, static variables etc. You must also make sure that your application
does not leak memory. Symfony itself is very good to handle this using
[resettable services](https://symfony.com/doc/current/reference/dic_tags.html#kernel-reset),
but your application may suffer from a memory leak. Just make sure your services
dont hold state and you will be fine.

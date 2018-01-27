---
title: Open files in PHPStorm from you Symfony application
author: Tobias Nyholm
date: '2015-04-15 08:55:00 +0200'
categories:
- Developing
- Symfony2
tags: []
---

If you want to open files in PHPStorm8 directly from the Symfony debug toolbar there is a neat trick you can use. This is very helpful when you quickly want to find a controller or when you got an exception.


{% highlight javascript %}
// /app/config/config.yml
framework:
  ide: &quot;phpstorm://open?file=%%f&amp;line=%%l&quot;
{% endhighlight %}


And that is pretty much it. You dont need to install PhpStormOpener, LinCastor or any other Apple script. This will work with PHPStorm8, Symfony2.6 and Mac OSX. Below is a picture to show where you should click to make PHPStorm open the controller used in this request.


![]({{ "/assets/images/phpstorm8_exception.png" | absolute_url }})
![PHPStorm8 Symfony debug toolbar]({{ "/assets/images/phpstorm8_debugtoolbar.png" | absolute_url }})



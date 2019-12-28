---
title: Open files in PHPStorm from you Symfony application
author: Tobias Nyholm
date: '2015-04-15 08:55:00 +0200'
categories:
- Developing
- Symfony
---

If you want to open files in PHPStorm8 directly from the Symfony debug toolbar there
is a neat trick you can use. This is very helpful when you quickly want to find 
a controller or when you got an exception.

{% highlight yaml %}
// /config/packages/framework.yml
framework:
  ide: "phpstorm://open?file=%%f&line=%%l"

{% endhighlight %}

You could also do this trick locally on your machine instead. That will not conflict
with other developers in your team and it will work for all your projects. 

{% highlight ini %}
; php.ini
xdebug.file_link_format='phpstorm://open?file=%f&line=%l'

; Alternative link syntax:
;xdebug.file_link_format='phpstorm://open?url=file://$file&line=$line'

{% endhighlight %}

And that is pretty much it. You dont need to install PhpStormOpener, LinCastor or
any other Apple script. This will work with PHPStorm8+, Symfony2.6+ and Mac OSX. 

Below is a picture to show where you should click to make PHPStorm open the controller
used in this request.

![]({{ "/images/posts/phpstorm8_exception.png" | absolute_url }})
![PHPStorm8 Symfony debug toolbar]({{ "/images/posts/phpstorm8_debugtoolbar.png" | absolute_url }})

Read more about this feature in the [Symfony documentation](https://symfony.com/doc/current/reference/configuration/framework.html#ide).

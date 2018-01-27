---

status: publish
published: true
title: Coveralls with Github and Travis
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 651
wordpress_url: http://developer.happyr.com/?p=651
date: '2014-07-15 09:19:38 +0200'
date_gmt: '2014-07-15 07:19:38 +0200'
categories:
- Developing
tags: []
comments:
- id: 77387
  author: devsdmf
  author_email: devsdmf@gmail.com
  author_url: http://www.devsdmf.net
  date: '2015-02-06 22:18:21 +0100'
  date_gmt: '2015-02-06 21:18:21 +0100'
  content: Very good!! This help me!!
- id: 107362
  author: bblue
  author_email: aleksander.lanes@gmail.com
  author_url: ''
  date: '2015-12-31 20:40:27 +0100'
  date_gmt: '2015-12-31 19:40:27 +0100'
  content: Will this work with the recent 1.0.0 release of satooshi/php-coveralls?
- id: 108236
  author: juniorRubyist
  author_email: geis28@gmail.com
  author_url: http://juniorRubyist.github.io
  date: '2016-03-03 18:44:49 +0100'
  date_gmt: '2016-03-03 17:44:49 +0100'
  content: I want to know, what is coverage? Like, what does Coveralls exactly tell
    you?
- id: 109342
  author: Dmitry
  author_email: madmis@inbox.ru
  author_url: http://madmis.com.ua
  date: '2016-06-02 07:39:53 +0200'
  date_gmt: '2016-06-02 05:39:53 +0200'
  content: Thank you man. You help me too.
- id: 109747
  author: Petr Kotek
  author_email: kotekp@gmail.com
  author_url: ''
  date: '2016-10-13 17:34:24 +0200'
  date_gmt: '2016-10-13 15:34:24 +0200'
  content: Thanks! Helpful even for v1.0.x
- id: 111009
  author: dfghj
  author_email: ghj@ueu.com
  author_url: http://wer.werrrewiqj
  date: '2017-03-21 13:27:07 +0100'
  date_gmt: '2017-03-21 12:27:07 +0100'
  content: Very nice comment.
- id: 112055
  author: Pavel
  author_email: zigzag.mcquack@gmail.com
  author_url: ''
  date: '2017-09-25 13:38:23 +0200'
  date_gmt: '2017-09-25 11:38:23 +0200'
  content: "Thx, Tobias!\r\n\r\nYour manual is more useful as the official. =) All
    works fine for me!"
---

The other day when I was making a PR to a Github project I got a comment like this:

<blockquote>
<p class="p1">Coverage decreased (-0.87%) when pulling <span class="s1"><b>f285fe0</b></span><b> on Nyholm:patch-1</b> into <span class="s1"><b>5506244</b></span><b> on some-project:master</b>.

</blockquote>
<p class="p1">This was an automated message from <a href="https://coveralls.io/">Coveralls.io</a>. It is a tool made for Ruby to keep track of your test code coverage. I think it is a nice way of telling the pull request author: "You need to add tests".

<p class="p1">If you want to install Coveralls to your PHP project there is a few step you need to do. But first of, make sure you use Github and Travis.

<h2 class="p1">Step 1 - Register your project at Coveralls</h2>

Go to <a href="https://coveralls.io/">Coveralls.io</a> and login with your Github account. You will get a list of your repositories (just like in Travis) and then select those you want to use with Coveralls.

<h2>Step 2 - Tell Travis to tell Coveralls</h2>

You need to tell Travis to send data to Coveralls. This is done with the satooshi/php-coveralls library. This is a example .travis.yml file


{% highlight javascript %}


language: php


php:
  - 5.3
  - 5.4
  - 5.5
  - 5.6
  - hhvm


install:
  - composer require satooshi/php-coveralls:~0.6@stable


before_script:
  - mkdir -p build/logs


script:
  - phpunit --coverage-clover build/logs/clover.xml


after_success:
  - sh -c 'if [ &quot;$TRAVIS_PHP_VERSION&quot; != &quot;hhvm&quot; ]; then php vendor/bin/coveralls -v; fi;'


{% endhighlight %}


I start by installing satooshi/php-coveralls. Then I make sure I got a folder to store the reports. I run PHPUnit and make sure it is generating some the reports. At last we run coveralls (but not on HHVM). Easy as pie =)

<h2>Step 3 - Optional</h2>

You may want to update you phpunit.xml.dist. Do this to make sure the proper files and directories are used when calculating the code coverage.  See this example:


{% highlight html %}
&lt;phpunit ...&gt;
  &lt;!-- Add a filter to make sure we don't count venders and Tests in the coverage report --&gt;
  &lt;filter&gt;
        &lt;whitelist&gt;
            &lt;directory suffix=&quot;.php&quot;&gt;./src&lt;/directory&gt;
            &lt;exclude&gt;
                &lt;directory&gt;./vendor&lt;/directory&gt;
                &lt;directory&gt;./tests&lt;/directory&gt;
            &lt;/exclude&gt;
        &lt;/whitelist&gt;
    &lt;/filter&gt;
&lt;/phpunit&gt;
{% endhighlight %}


That is all you need to do. Happy testing!


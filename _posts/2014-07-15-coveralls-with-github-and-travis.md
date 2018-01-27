---
title: Coveralls with Github and Travis
author: Tobias Nyholm
date: '2014-07-15 09:19:38 +0200'
categories:
- Developing
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


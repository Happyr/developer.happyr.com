---

status: publish
published: true
title: Using Bower for your Symfony2 application
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 306
wordpress_url: http://developer.happyr.com/?p=306
date: '2013-10-15 21:36:55 +0200'
date_gmt: '2013-10-15 19:36:55 +0200'
categories:
- Symfony2
- Bower
tags: []
comments:
- id: 70591
  author: Tommy
  author_email: pusle@pusle.com
  author_url: ''
  date: '2014-12-29 20:48:04 +0100'
  date_gmt: '2014-12-29 19:48:04 +0100'
  content: Thank you for this, I find it cleaner to seperate front-end from back-end.
    Using bower bundles in symfony just don't feel right.
- id: 109872
  author: Gintare
  author_email: g.statkute@gmail.com
  author_url: ''
  date: '2016-11-01 15:53:10 +0100'
  date_gmt: '2016-11-01 14:53:10 +0100'
  content: Thank you. It was useful.
- id: 110428
  author: Edgar KAMDEM
  author_email: briceouabo@gmail.com
  author_url: ''
  date: '2016-12-27 13:43:46 +0100'
  date_gmt: '2016-12-27 12:43:46 +0100'
  content: Thx Tobias, very useful.
- id: 111340
  author: Richard
  author_email: rileyrg@gmail.com
  author_url: ''
  date: '2017-04-27 06:59:27 +0200'
  date_gmt: '2017-04-27 04:59:27 +0200'
  content: A bit late to the party but where does the .bowerrc go in a symfony project?
---

I recently switched to Bower for managing my frontend third party libraries. First I defined some packages that I needed in a bower.json file and put it in the Symfony2 root directory.


{% highlight javascript %}


{
&quot;name&quot;: &quot;MyApp&quot;,
&quot;version&quot;: &quot;0.1.0&quot;,
&quot;dependencies&quot;: {
&quot;jquery&quot;: &quot;1.10.2&quot;,
&quot;Ink&quot;: &quot;2.2.1&quot;,
&quot;select2&quot;: &quot;3.4.3&quot;,
&quot;jquery.cookie&quot;: &quot;1.4.0&quot;
}
}


{% endhighlight %}


Then I told Bower to put the downloaded files in my <em>/web</em> folder. The /web folder makes sense because the <em>/vendor</em> is owned my Composer and the <em>/app/Resources/public</em> is not made for application wide assets like <em>@Bundle/Resources/public</em> is. If you put contents in <em>/app/Resources/public</em> it will not be moved when you do an <strong>assets:install</strong> or <strong>assetic:dump</strong>.


An other reason why you should put the files in<em> /web</em> is that the CSS directive @import will not work otherwise. The cssrewrite filter can't manage to get the correct URLs... so always put you bower components in the web folder.


To tell Bower to put the downloaded files in <em>/web</em>  you need to create a .bowerrc file next to bower.json with the following contents:


{% highlight javascript %}


{
&quot;directory&quot;: &quot;web/bower&quot;,
&quot;json&quot;: &quot;bower.json&quot;
}


{% endhighlight %}


After you do a <strong>bower install</strong> you may use your new libraries by including them from any twig like:


{% highlight html %}


\{\% javascripts
'bower/jquery/jquery.min.js'
'bower/Ink/js/holder.js'
'bower/Ink/js/ink.min.js'
'bower/Ink/js/ink-ui.min.js'
'bower/Ink/js/autoload.js'
'bower/Ink/js/html5shiv.js'
'bower/select2/select2.min.js'
'bower/select2/select2_locale_sv.js'
'bower/jquery.cookie/jquery.cookie.js'
\%\}
 &lt;script type=&quot;text/javascript&quot; src=&quot;{{ asset_url }}&quot;&gt;&lt;/script&gt;
 \{\% endjavascripts \%\}


{% endhighlight %}


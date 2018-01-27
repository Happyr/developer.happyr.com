---
title: Using Bower for your Symfony2 application
author: Tobias Nyholm
date: '2013-10-15 21:36:55 +0200'
categories:
- Symfony2
- Bower
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


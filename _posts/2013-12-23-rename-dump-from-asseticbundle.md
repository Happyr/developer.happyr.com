---

status: publish
published: true
title: Rename dump from AsseticBundle
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 350
wordpress_url: http://developer.happyr.com/?p=350
date: '2013-12-23 11:35:54 +0100'
date_gmt: '2013-12-23 10:35:54 +0100'
categories:
- Symfony2
tags: []
comments:
- id: 69
  author: Marcus
  author_email: marcus.malka@gmail.com
  author_url: ''
  date: '2014-01-14 14:52:38 +0100'
  date_gmt: '2014-01-14 13:52:38 +0100'
  content: "Thanks for this, this was a useful reference.\r\n\r\nI implemented a similar
    system that creates version.yml dynamically. In our case the assetic_version is
    fetched from our automatically created git tag."
---

In your main template you <span style="text-decoration: line-through;">probably</span> should have assetic to manage your assets. In the production environment you would like to dump this to static files.


{% highlight html %}


//base.html.twig


\{\% javascripts
 'bower/jquery/jquery.min.js'
 '@AcmeDemoBundle/Resources/public/js/*'
 '@AcmeBlogBundle/Resources/public/js/*'
\%\}
 &lt;script type=&quot;text/javascript&quot; src=&quot;{{ asset_url }}&quot;&gt;&lt;/script&gt;
 \{\% endjavascripts \%\}


{% endhighlight %}


To dump the assets you run the following command:


{% highlight bash %}php app/console assetic:dump --env=prod --no-debug{% endhighlight %}


This will generate a file like <em>root/web/js/53ad2515.js</em>. This is awesome. This file should be cached with very generous expiration date. This file should also be handled by your CDN.


The problem is when you want to deploy a new version of your application, with lots of changes in your javascripts. The assetic:dump command will generate the <em>root/web/js/53ad2515.js</em> again which leads to that your returning visitor does not get the latest version of the file.

<h2>Symfony's solution</h2>

The solution suggested by the <a href="http://symfony.com/doc/current/reference/configuration/framework.html#ref-framework-assets-version">Symfony2 docs</a> is that you use <em>assets_version</em>. You should update that config every time you made some changes in the assets. But that gives a new version of <strong>all</strong> the assets, including images, and forces my users to download them again.

<h2>My solution</h2>

I would like my users to download some assets every time I deploy. I start with specifying an output in my main template.


{% highlight html %}


//base.html.twig


\{\% javascripts output='js/scripts.{version}.js'
'bower/jquery/jquery.min.js'
'@AcmeDemoBundle/Resources/public/js/*'
'@AcmeBlogBundle/Resources/public/js/*'
 vars=[&quot;version&quot;]
\%\}
&lt;script type=&quot;text/javascript&quot; src=&quot;{{ asset_url }}&quot;&gt;&lt;/script&gt;
\{\% endjavascripts \%\}


{% endhighlight %}


Since Assetic needs to precompile the assets it needs to know what the available values <em>version</em> may have. We just need one version.


{% highlight html %}
# app/config/config.yml
# Assetic Configuration
assetic:
  variables:
    version: [%git_commit%]


{% endhighlight %}


I use the parameter <em>%git_commit%</em>. I want this parameter to change every time I deploy. Create a new file called <em>app/config/version.yml</em> and make sure to include it in <em>app/config/config.yml</em>.


{% highlight html %}
# app/config/config.yml
imports:
  - { resource: parameters.yml }
  - { resource: version.yml }
{% endhighlight %}


{% highlight html %}
# app/config/version.yml
parameters:
 git_commit: 'sha1'
{% endhighlight %}


You just have to do one more thing before this will work. You have to "give" that variable value to Assetic. You have to override the AsseticBundle\DefaultValueSupplier with a class of your own.


{% highlight html %}
#AcmeDemoBundle/Resources/config/services.yml
parameters:
  assetic.value_supplier.class: Acme\DemoBundle\Assetic\CustomValueSupplier
{% endhighlight %}
{% highlight php %}
&lt;?php
// src/Acme/DemoBundle/Assetic/CustomValueSupplier.php
namespace Acme\DemoBundle\Assetic;


use Symfony\Bundle\AsseticBundle\DefaultValueSupplier;


class CustomValueSupplier extends DefaultValueSupplier
{
    /**
     * Get values for Assetic
     *
     * @return array
     */
    public function getValues()
    {
        //get the default values
        $values = parent::getValues();


        //get the git version as version
        $values['version']=$this-&gt;container-&gt;getParameter('git_commit');


        return $values;
    }
}
{% endhighlight %}


If you run <em>assetic:dump</em> now you will get a file called <em>root/web/js/script.sha1.js</em>. The tricky part now is to automatically change version.yml when you deploy. I make sure my deployment script executes the following bash script.


{% highlight bash %}
#!/bin/bash


FILE=&quot;app/config/version.yml&quot;


echo &quot;#This is an auto generated file that will be updated at every deploy&quot; &gt; $FILE


echo &quot;parameters:
 git_commit: '$(git rev-parse --short HEAD)'&quot; &gt;&gt; $FILE


{% endhighlight %}


With this little configuration I make sure that my JavaScripts and CSS are always up to date without me having to remember to bump the version in any configuration file.


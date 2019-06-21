---
title: Using NewRelic with Bref on AWS Lambda
author: Tobias Nyholm
date: '2019-06-21 18:18:47 +0200'
header:
  image: "images/posts/non-blocking-fonts.png"
  teaser: images/posts/non-blocking-fonts.png
categories:
- Bref
---

Running serverless is great. When migrating our applications to use Bref we realized that we need a non-standard php-extension. 
With "non-standard" I mean that the extension is not included by default by Bref. With some help I manage to make every 
thing work as I wanted it to. Here is what I did: 

I cloned the Bref repository to be able to build my own image. Then I added the NewRelic extension and config to `php.Dockerfile`
in the /runtime/php directory. 

{% highlight docker %}
## Install NewRelic

RUN \
  curl -L https://download.newrelic.com/php_agent/release/newrelic-php5-8.7.0.242-linux.tar.gz | tar -C /tmp -zx && \
  export NR_INSTALL_USE_CP_NOT_LN=1 && \
  export NR_INSTALL_DAEMONPATH=${INSTALL_DIR}/sbin/newrelic-daemon && \
  export NR_INSTALL_SILENT=1 && \
  /tmp/newrelic-php5-*/newrelic-install install && \
  rm -rf /tmp/newrelic-php5-* /tmp/nrinstall*

RUN echo $' \n\
extension = "newrelic.so" \n\
newrelic.appname = "BrefLambda" \n\
newrelic.license = "0123456789" \n\
newrelic.logfile = "/dev/null" \n\
newrelic.loglevel = "error" \n\
' >> ${INSTALL_DIR}/etc/php/php.ini

RUN mkdir -p ${INSTALL_DIR}/etc/newrelic && \
  echo "loglevel=error" > ${INSTALL_DIR}/etc/newrelic/newrelic.cfg && \
  echo "logfile=/dev/null" >> ${INSTALL_DIR}/etc/newrelic/newrelic.cfg

{% endhighlight %}

I disable logging since I run this on a read-only environment.  

This is great and it should have been working on a normal server. However, NewRelic is configured to automatically start
with the PHP-FPM server. Since Bref has it custom implementation of PHP-FPM that will not work. We need to start the the
newrelic-daemon manually every time we load or Lambda function. 

Let's start with the `function` layer: 

{% highlight docker %}
# runtime/php/layers/function/bootstrap

# Start NewRelic daemon
/opt/bref/sbin/newrelic-daemon -c /opt/bref/etc/newrelic/newrelic.cfg
{% endhighlight %}

Finally done, but let's move on to the `website` layer. First I rename the `boostrap` file to `bootstrap-php`. Then I create
a new `bootstrap` file with the following contents: 

{% highlight shell %}
#!/bin/sh

# Start NewRelic daemon
/opt/bref/sbin/newrelic-daemon -c /opt/bref/etc/newrelic/newrelic.cfg

# Start PHP-FPM
/opt/bootstrap-php
{% endhighlight %}

Now I add a line in `bootstrap-php` to make sure NewRelic is not consider my requests as background jobs:

{% highlight php %}
// bootstrap-php

// ...
ini_set('display_errors', '1');
error_reporting(E_ALL);

// Tell NewRelic that this is not a background job
newrelic_background_job(false);

// ...
{% endhighlight %}

Last thing to do is to make sure the new `bootstrap-php` is part of the runtime zip file. 

{% highlight shell %}
# export.sh

# ...

# Create the PHP FPM layer
# Add files specific to this layer
cp /layers/fpm/bootstrap bootstrap
cp /layers/fpm/bootstrap-php bootstrap-php   # <---- New line
chmod 755 bootstrap bootstrap-php            # <---- Modified line
cp /layers/fpm/php.ini bref/etc/php/conf.d/bref.ini
cp /layers/fpm/php-fpm.conf bref/etc/php-fpm.conf
# Zip the layer
zip --quiet --recurse-paths /export/php-${PHP_SHORT_VERSION}-fpm.zip . --exclude "*php-cgi"
# Cleanup the files specific to this layer
rm bootstrap bootstrap-php                   # <---- Modified line
rm bref/etc/php/conf.d/bref.ini
rm bref/etc/php-fpm.conf
{% endhighlight %}

Now we are finally done. Just rebuild your php docker, package your zip files and publish them on AWS. 

To accomplish this took me way more hours then I want to admit. So I hope this small post could help someone to be a litte
faster than I was. 

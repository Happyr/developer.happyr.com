---
title: Compiling PHP 7.4 on OSX
author: Tobias Nyholm
date: '2020-01-12 11:18:00 +0200'
header: 
  image: "images/posts/php.jpg"
  teaser: images/posts/php.jpg
categories:
- PHP
--- 

I think I am a decent PHP programmer, but I am lost when it comes to devops, C,
Makefiles and compiling stuff. I just want to write ``brew upgrade php@74`` and
be happy. If that doesn't work I have to google each small step to fix the issue. 

I love writing bash for example, but I need to google the syntax of an if-statement
for every bash script I write. 

The RC for PHP 7.4.2 just came out this week. I decided to test them out. I downloaded
them from the website http://qa.php.net/, unzipped them. I carefully followed some guide
saying: 


{% highlight cli %}
./configure
make
make install

{% endhighlight %}

If course I run in to issues like: 

{% highlight cli %}
> #include <libxml/parser.h>
> fatal error: 'libxml/parser.h' file not found

{% endhighlight %}

Since OSX dont have standard paths where libraries are you need to provide them to the 
``.configure`` call. In OSX version before Catalina (10.15) you could do some [trick](https://silvae86.github.io/sysadmin/mac/osx/mojave/beta/libxml2/2018/07/05/fixing-missing-headers-for-homebrew-in-mac-osx-mojave/), 
but that does not work any more.  

And in PHP versions before 7.4 you could options like ``--with-libxml-dir``, but that is 
[removed](https://externals.io/message/107846) in favor of ``pkg-config``.

## What you should do

So every time it complains, ie for libxml2, you should update the `PGK_CONFIG_PATH`
environment variable. The value depends on the path to the library.
 
To find the path, lets run ``brew info``
 
{% highlight cli %}
➜  brew info libxml2
libxml2: stable 2.9.10 (bottled), HEAD [keg-only]
GNOME XML library
http://xmlsoft.org/
/usr/local/Cellar/libxml2/2.9.9_2 (281 files, 10.5MB)
  Poured from bottle on 2019-02-26 at 23:49:05
From: https://github.com/Homebrew/homebrew-core/blob/master/Formula/libxml2.rb
==> Dependencies
Required: python ✘, readline ✔
==> Options
--HEAD
	Install HEAD version
==> Caveats
libxml2 is keg-only, which means it was not symlinked into /usr/local,
because macOS already provides this software and installing another version in
parallel can cause all kinds of trouble.

==> Analytics
install: 40,738 (30 days), 149,637 (90 days), 604,534 (365 days)
install-on-request: 11,176 (30 days), 39,774 (90 days), 130,051 (365 days)
build-error: 0 (30 days)

{% endhighlight %}

From the output we see that libxml2 lives in ``/usr/local/Cellar/libxml2/2.9.9_2``.
That means pkgconfig lives in this folder:

{% highlight cli %}
PKG_CONFIG_PATH=/usr/local/Cellar/libxml2/2.9.9_2/lib/pkgconfig

{% endhighlight %}

For my machine, I had to run the following commands to successfully compile PHP.

{% highlight cli %}

env PKG_CONFIG_PATH=/usr/local/Cellar/openssl@1.1/1.1.1d/lib/pkgconfig:/usr/local/Cellar/libxml2/2.9.9_2/lib/pkgconfig:/usr/local/Cellar/icu4c/64.2/lib/pkgconfig \
      ./configure 
      --prefix /usr/local/Cellar/php/7.4.2RC1 \
      --sysconfdir=/usr/local/Cellar/php/7.4.2RC1 \
      --with-config-file-path=/usr/local/etc/php/7.4.2RC1 \
      --with-config-file-scan-dir=/usr/local/etc/php/7.4.2RC1/conf.d \
      --enable-bcmath \
      --enable-calendar \
      --enable-dba \
      --enable-dtrace  \
      --enable-exif \
      --enable-ftp \
      --enable-fpm \
      --enable-gd \
      --enable-intl \
      --enable-mbregex \
      --enable-mbstring \
      --enable-mysqlnd \
      --enable-pcntl \
      --enable-phpdbg \
      --enable-phpdbg-webhelper \
      --enable-shmop \
      --enable-soap \
      --enable-sockets \
      --enable-sysvmsg \
      --enable-sysvsem \
      --enable-sysvshm \
      --with-curl \
      --with-fpm-user=_www \
      --with-fpm-group=_www \
      --with-freetype \
      --with-iconv=/usr/local/Cellar/libiconv/1.16 \
      --with-libxml \
      --with-jpeg \
      --with-layout=GNU \
      --with-mysql-sock=/tmp/mysql.sock \
      --with-openssl \
      --with-pdo-mysql=mysqlnd \
      --with-pdo-pgsql=/usr/local/Cellar/libpq/12.1_1 \
      --with-pdo-sqlite \
      --with-pgsql=/usr/local/Cellar/libpq/12.1_1 \
      --with-pic \
      --with-sodium \
      --with-sqlite3 \
      --with-unixODBC \
      --with-webp \
      --with-xmlrpc \
      --with-xsl \
      --with-zip \
      --with-zlib

make
make install

{% endhighlight %}

I struggled for hours to compile PHP 7.4. My main mistake was that I didn't read the
output of the ``./configure`` command. Please please please, make sure to read at least
the last 20 lines. 
 
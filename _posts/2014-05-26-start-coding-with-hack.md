---
title: Start coding with Hack
author: Tobias Nyholm
date: '2014-05-26 15:15:39 +0200'
categories:
- HHVM
- Hack
tags: []
---

If you like to take your PHP to the next level there is an excellent option for you: Hack. It is a programming language develop by Facebook. They have been using it for a couple of years now. In the spring of 2014 they decided to remove all ”Facebook stuff” from Hack and release the language as open source.


So you are probably wondering why Hack is a good thing. What can Hack do that PHP can’t? The truth is that it is not much Hack can do that PHP can’t… But there are some features that will make you a better developer. Hack is a static typed language. That means that you have to declare a type of each variable, function parameter and return values. Like Java or C#. This is good because it helps you find bugs while you are writing the code!

<h2>Install Hack</h2>

Hack does only run on HHVM. It is also developed by Facebook and will run Hack and PHP really fast! If you want to test Hack I recommend using vagrant and <a href="https://github.com/Nyholm/vagrant-hhvm">this repo</a>.

<ol>
<li>Install <a href="https://www.virtualbox.org/">VirtualBox</a></li>
<li>Install <a href="http://docs.vagrantup.com/v2/installation/">Vagrant</a></li>
<li>Clone the repo into /some/path/hhvm</li>
<li>Edit your /etc/hosts file to include <em>hhvm.local</em> with IP <em>192.168.99.99</em></li>
<li>Open a terminal and go to /some/path/hhvm</li>
<li>Run <strong>vagrant up</strong> (it will take a while)</li>
</ol>

When the <strong>vagrant up</strong> has finished downloading the virtual machine and installed everything, go to http://hhvm.local in your browser. If you see “Hello Hack!” you know everything works.


Look at the source folder. You find a file called index.hh. It contains a very complicated way of printing a text string. I’ve written it like that to show you about generic types. You may make a change to the index.hh and then see your changes in the browser directly. In this way, Hack works just like PHP. You don't need to compile anything.

<h2>Finding bugs</h2>

I said Hack was great to find bugs.With some help from the command hh_client you will get detailed error messages when you’ve done something wrong.


{% highlight bash %}
$ cd /some/path/hhvm
$ vagrant ssh
$ cd /vagrant/source
$ hh_client
No errors!
$ nano index.hh
# change “getAStore(): Store&lt;string&gt;” to “getAStore(): Store&lt;int&gt;”
$ hh_client
 /vagrant/source/index.hh:23:12,13: Invalid return type
 /vagrant/source/index.hh:19:29,31: This is an int
 /vagrant/source/index.hh:21:13,25: It is incompatible with a string
{% endhighlight %}


What happened here is that we said getAStore should return a Store of integers but instead we returned a Store of strings. Hack sees that we have done something wrong and gives us a warning.


There are editor plugins for hh_client for Vim and Emacs. It’s only a matter of time until someone builds plugins for sublime or PHPStorm.


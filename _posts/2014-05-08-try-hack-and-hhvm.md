---
title: Try Hack and HHVM
author: Tobias Nyholm
date: '2014-05-08 12:17:24 +0200'
categories:
- HHVM
- Hack
---

Lately there has been a lot of talk about HHVM and the new programming language Hack. It all looks great, to bad it does not run your code. The HHVM team is doing their very best to make it 100% backwards compatible with PHP. So at the moment there is very few project that can run on HHVM. But that should not stop you from learning all about HHVM and Hack.


If you want to get your hands dirty with HHVM and Hack the easiest way is to use a virtual machine, Vagrant and Puppet. Don't be alarmed if you haven't heard of these techniques before. They are built to make your life simpler.


The first thing you got to do is make sure you have <a style="color: #4183c4;" href="https://www.virtualbox.org/">VirtualBox</a> installed and then <a style="color: #4183c4;" href="http://docs.vagrantup.com/v2/installation/">install Vagrant</a>. The second thing you need to do is to clone <a href="https://github.com/Nyholm/vagrant-hhvm">this repository</a>. It contains scripts (recipes) how the virtual machine should be configured. Open the terminal and go to the unzipped folder (ie <em>/Users/tobias/Desktop/hhvm</em>) and run <strong>vagrant up</strong> to start the virtual machine. The last thing you need to do is to add a line in your /etc/hosts like:


{% highlight bash %}#/etc/hosts
192.168.99.99 hhvm.local sf.hhvm{% endhighlight %}


You should now go to your browser and type the url <a href="http://hhvm.local" target="_blank">http://hhvm.local</a> and see the "Hello Hack!" message. The source of the script is in the <em>source</em> directory (ie <em>/Users/tobias/Desktop/hhvm/source</em>). Edit the file and reload your browser to see the changes. Happy Hacking!

<h2>Running Symfony2 project on HHVM</h2>

If you want to try running an existing project of yours on HHVM. Uncomment the last part in the Vagrant file and  update the path:


{% highlight bash %}# If you want to run a symfony application uncomment this line and change the path
config.vm.synced_folder &quot;/Users/tobias/Desktop/MyExistingSymfonyProject&quot;, &quot;/srv/sf.hhvm/&quot;{% endhighlight %}


If you run <strong>vagrant provision</strong> you will update the machine and you will be able to go to <a href="http://sf.hhvm" target="_blank">http://sf.hhvm</a>  in your browser to run your symfony2 project.


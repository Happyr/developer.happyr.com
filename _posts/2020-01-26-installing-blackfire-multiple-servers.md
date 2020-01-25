---
title: Installing Blackfire on multiple servers
author: Tobias Nyholm
date: '2020-01-26 16:21:00 +0200'
header:
  image: "images/posts/blackfire/header.png"
  teaser: images/posts/blackfire/header.png
categories:
- PHP
- Blackfire
---

At Happyr, we have about 30 applications running across 14 Amazon EC2 servers and 
a lot of Lambda functions. Most of them are small microservice applications that 
are not regularly updated with new features. I've installed and configured so I 
can run Blackfire's profiler on each of these applications. Blackfire's internal
architecture allows me to install the profiler in a bit more clever way, compared
to other profilers I've tried. 

This article will go over how to install Blackfire when you have multiple servers
and give very concrete examples with Amazon. I'm extra excited for this tutorial
because it will work perfectly with the upcoming [Blackfire APM](https://hello.blackfire.io/apm).

## The basics

In the [Blackfire installation documentation](https://blackfire.io/docs/up-and-running/installation)
you can read that Blackfire consists of 4 parts. First of course are the *Blackfire server*. 
They receive all data and creates a nice dashboard for you. 

There is also an *Agent* than runs on our machine that prepare and [cleans the data](https://blackfire.io/docs/reference-guide/faq#what-data-is-sent-to-the-blackfire-servers) 
from sensitive information and sends it to the Blackfire servers. 

Then there is a *PHP extension* that collects the metrics from your application
and sends it to the *Agent*.

Finally there is a *Client* that can start the profiling. There are two clients
that you are most likely to use; a browser extension and a CLI tool. The client
is used to make sure that only authorized users (you) can start profiling. 

## Install

When installing Blackfire on multiple servers, you dont need to have multiple instances
of the Agent. You may have one agent for all your application. You just need to
install the PHP extension on all servers and configure the extension with the
IP address to the machine that is running the Agent. 

### Installing the agent

Login to your [AWS Console](https://console.aws.amazon.com/), navigate to EC2
and create a new instance. The first step is to select an AMI. Instead of selecting
Ubuntu or something similar, search for "Blackfire" in the Community AMIs and select
"Blackfire.io Agent".

![Select AMI]({{ "/images/posts/blackfire/select-ami.png" | absolute_url }})

Then it is time to select the instance type. The agent does not need much computer
power, running with t2.micro is perfect. 

![Select instance type]({{ "/images/posts/blackfire/select-instance-type.png" | absolute_url }})

Review and launch your instance. Use a SSH key pair that you that you have access to
because next step is to SSH into the machine. 

![Review and Launch]({{ "/images/posts/blackfire/review.png" | absolute_url }})

Now we wait a minute for the machine to be ready...

When instance state is "running" we can SSH into the machine. Find the machines
public IP address. The instance in the screenshot below has public IP 54.172.34.219
and private IP 172.31.84.248.

![Instance example]({{ "/images/posts/blackfire/instance-show.png" | absolute_url }})

We need to SSH into the Agent instance to update it and configure it with the server 
token related to your Blackfire account. You find the correct command in the section 
"Configure the Agent" in the [Blackfire installation guide](https://blackfire.io/docs/up-and-running/installation). 

![Login to agent instance]({{ "/images/posts/blackfire/login-to-agent.png" | absolute_url }})
 
{% highlight cli %}
sudo apt-get uppdate
sudo apt-get upgrade

{% endhighlight %}

{% highlight cli %}
sudo blackfire-agent --register --server-id=xxxxxxxx --server-token=yyyyyyy
sudo /etc/init.d/blackfire-agent restart 

{% endhighlight %}

The agent is already configured to accept connections from any IP on port 8307.
This is safe as long as port 8307 is not accessible from the internet. (Only port
22 is accessible from the internet by default.)   

We are now done, with the agent. Now we need to install the PHP extension and 
configure it to send data to the agent on the private IP 172.31.84.248 and port 8307.

### Installing the PHP extension

There is nothing really special when installing the agent. The 
[Blackfire installation guide](https://blackfire.io/docs/up-and-running/installation)
covers this really well. 

If you are using [Bref](https://bref.sh/) to deploy your application on AWL Lambda
then you maybe interested in [this repository](https://github.com/brefphp/extra-php-extensions) 
to install the Blackfire extension.

### Configure the PHP extension

We need to modify *php.ini* to add the following parameters: 

{% highlight ini %}
blackfire.agent_socket = tcp://172.31.84.248:8307
blackfire.agent_timeout = 0.25

{% endhighlight %}

And of course restart PHP-FPM to make the changes visible.

## Try it out

Now you only need to install a client (eg Blackfire browser extension) and you
are ready to profile your production applications. 

## Troubleshoot

If it does not work for you. Make sure that you application has connectivity to
the Agent. The easiest way is to put the agent in the same VPC as your application, 
then create a Security Group that allows communication to the agent's port 8307
from the application. 

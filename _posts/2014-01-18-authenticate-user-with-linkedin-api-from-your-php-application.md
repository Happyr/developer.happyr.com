---
title: Authenticate user with LinkedIn API from your PHP application
author: Tobias Nyholm
date: '2014-01-18 13:26:05 +0100'
categories:
- Library
tags: []
---

Recently I wanted to let my users authenticate with LinkedIn. We had just added a Facebook and Twitter authentication, so why not LinkedIn. To my big surprise there were no <strong>LinekdIn API clients for PHP</strong>. At least no good ones. And the code samples from <a href="https://developer.linkedin.com/documents/code-samples">developer.linkedin.com</a> where lookting pretty nasty. So, as a good open source developer I decided to build one and share it.

<h2>My LinkedIn API client</h2>

With inspiration from the Facebook PHP SDK I came up with a solution that hides all the nasty stuff from the code samples. Click here to review and download my <a href="https://github.com/HappyR/LinkedIn-API-client">LinkedIn PHP Api client</a>. I wanted to focus on these three things:

<ol>
<li>Easy to implement</li>
<li>Easy to extend</li>
<li>Easy to test</li>
</ol>

So how do you use it? Study this code example and tell me in the comments what you think:


{% highlight php %}
&lt;?php
//index.php


//make sure you have included composers autoload.php before you begin.
$linkedIn=new HappyR\LinkedIn\LinkedIn('app_id', 'app_secret');


//if not authenticated
if (!$linkedIn-&gt;isAuthenticated()) {
    $url = $linkedIn-&gt;getLoginUrl();
    echo &quot;&lt;a href='$url'&gt;Login with LinkedIn&lt;/a&gt;&quot;;
    exit();
}


//we know that the user is authenticated now
$user=$linkedIn-&gt;get('v1/people/~:(firstName,lastName)');


echo &quot;Welcome &quot;.$user['firstName'];


{% endhighlight %}


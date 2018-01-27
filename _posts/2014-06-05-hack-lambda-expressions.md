---

status: publish
published: true
title: Hack Lambda expressions
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 580
wordpress_url: http://developer.happyr.com/?p=580
date: '2014-06-05 06:13:07 +0200'
date_gmt: '2014-06-05 04:13:07 +0200'
categories:
- Happyr
tags: []
comments: []
---

Hack is all about making it easier for the developer to write good code. There has been some improvement of the PHP anonymous function or closer as they also are called.
An anonymous function is a function without a name (dah!). You may use them as a parameter to some native functions.


{% highlight php %}
$arr = array('hello', 'bazbar', 'foo');
usort($arr, function($a, $b) {
	return strlen($a)-strlen($b);
});


//$arr = array('foo', 'hello', 'bazbar');
{% endhighlight %}


As you can see there is some overhead writing this. Hack has made it easier with something they call a Lambda expression.


{% highlight php %}
$arr = array('hello', 'bazbar', 'foo');
usort($arr, ($a, $b) ==&gt; strlen($a)-strlen($b));


//$arr = array('foo', 'hello', 'bazbar');
{% endhighlight %}


This is just some syntactic sugar. The Facebook developers noticed that most of their anonymous functions had just one parameter and was only one or two lines. This new syntax will make the code more readable and cleaner.


There is also a feature here to be aware of. When using anonymous functions in you get a new scope where your variables live (the same behaviour as any other new function you create). If you want to use a variable outside the anonymous function you have to use the use statement.


{% highlight php %}
$x=2;
$result = array_map(function($a) use ($x) {
	return $a+$x;
}, array(1,2,3));


//$result = array(3,4,5);
{% endhighlight %}


With the Lambda expression (==>) you will inherit the scope from the ancestor.


{% highlight php %}
$x=2;
$result = array_map($a ==&gt; $a + $x, array(1,2,3));


//$result = array(3,4,5);
{% endhighlight %}

<h2>Something to be aware of</h2>

There are some things to be aware of here. The inherited scope is copied by value. So you cannot change any variables outside of the lambda expression.


{% highlight php %}
$x=2;
$result = array_map($a ==&gt; $a + $x++, array(1,2,3));


//$result = array(3,4,5);
{% endhighlight %}

<h2>More parameters or lines?</h2>

You may write a lambda expression wit multiple lines and/or parameters. You just wrap the parameters with parentheses and the lamda body with curly brackets. In my opinion you, if you are doing lambda expression with more than two lines you might as well use anonymous functions. Because long lambda expression will no longer be easy to read and anonymous functions allows you to use type annotation.


{% highlight php %}
$arr = array('hello', 'bazbar', 'foo');


usort($arr, ($a, $b) ==&gt; {
	$x=strlen($a);
	$y=strlen($b);
	return $x-$y;
});


//$arr = array('foo', 'hello', 'bazbar');
{% endhighlight %}


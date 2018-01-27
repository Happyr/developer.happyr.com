---

status: publish
published: true
title: Hack types
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 534
wordpress_url: http://developer.happyr.com/?p=534
date: '2014-05-28 12:06:05 +0200'
date_gmt: '2014-05-28 10:06:05 +0200'
categories:
- Hack
tags: []
comments:
- id: 845
  author: Joel Clermont
  author_email: jclermont@gmail.com
  author_url: http://joelclermont.com
  date: '2014-07-10 00:07:52 +0200'
  date_gmt: '2014-07-09 22:07:52 +0200'
  content: "<blockquote>\r\nThe biggest difference between Hack and PHP is that PHP
    is using dynamic typing and Hack is statically typed. Java and C# is other languages
    that is statically typed. What this means is that you have to declare a type on
    each variable.\r\n</blockquote>\r\n\r\nI'd clarify the \"you\" in that last statement.
    Static typing doesn't require the programmer to decorate every variable or function
    with a type. If the language has a strong type system, it can often infer types
    without any additional work by the programmer. Haskell, for example.\r\n\r\nI
    know very little about Hack, but it looks like it will infer types on local vars,
    but requires you to manually annotate them for function definitions and class
    vars."
- id: 864
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2014-07-10 08:38:27 +0200'
  date_gmt: '2014-07-10 06:38:27 +0200'
  content: "You're right. Static typed means that the variable is bound to a type
    and you may not change it. How you declare the type is not important to the definition.
    \r\n\r\nI'll rewrite that paragraph. Thank you for pointing that out."
- id: 77057
  author: Aftab Naveed
  author_email: aftabnaveed@gmail.com
  author_url: ''
  date: '2015-02-04 06:15:14 +0100'
  date_gmt: '2015-02-04 05:15:14 +0100'
  content: "@Joel Clermont\tyep that's right they infer type checking on function
    level, they think it is unnecessary, IMHO it just went on top of my head,  cause
    a local function variable can hold any type of value doesn't make sense, you can
    find my issue here \r\n\r\nhttps://github.com/facebook/hhvm/issues/4750"
---

The biggest difference between Hack and PHP is that PHP is using dynamic typing and Hack is statically typed. Java and C# is other languages that is statically typed. This means that the type on variables must be defined before runtime. Example of types are string, int, bool, array. There are even types like MyObject, Vector&lt;int&gt; and mixed. It is good practice to annotate all class variables, function parameters and function return types. There is no need to annotate local variables.


{% highlight php %}
&lt;?hh
class Foo {
  public function bar (int $var): string {
    return &quot;Number: &quot;.$var;
  }
}
{% endhighlight %}

<h2>So why types?</h2>

I order to find bugs before runtime you need types. It is very hard to do a type check before run time on a dynamically typed language. You may rely on a function foobar() returning an array but sometimes it might return null instead of an empty array. Or maybe it returns a scalar instead of an array with one element. When you are using a static typed language the type checker (or complier for Java and C#) will find that foobar() may return <em>null</em> but has declared <em>array</em> as return value.


The type checker will make your code reliable. You fill find that the contract between classes is much more strict. If the return type is declared as <em>array</em> you know that it will return an array. With PHP you could only <strong>be sure</strong> that you would get an array and you would have to read the production logs to find out when it doesn’t.


To activate the type checker you need to start your hack file with &lt;?hh instead of &lt;?php. There are 3 modes of the type checker: strict, partial and decl. The default mode is partial. I’ve written another blog post about the <a href="http://developer.happyr.com/hack-modes">modes and the differences</a>.

<h2>Type casting</h2>

In PHP you may write


{% highlight php %}
&lt;?php
class Foo {
  private $a=4711;   //$a is a int
  public function __contruct() {
    $this-&gt;a=&quot;foobar&quot;; // $a is a string
  }
}
{% endhighlight %}


That is not allowed in Hack:


{% highlight php %}
&lt;?hh
class Foo {
  private int $a=4711;   //$a is a int
  public function __contruct() {
    //You cant assign a string to an int
    $this-&gt;a=&quot;foobar&quot;; // $a is a string
  }
}
{% endhighlight %}


If you want to change a type you may use type casing:


{% highlight php %}
&lt;?hh
class Foo {
  private int $a=4711;
  private bool $b;
  private bool $c;
  public function __contruct() {
    //You cant assign a string to an int
    $this-&gt;b = (bool) $this-&gt;a;
    $this-&gt;c = (string) $this-&gt;a;
  }
}
{% endhighlight %}


Hack has chosen consistency over convenience when it comes to type aliasing. You may not use integer over int, Boolean over bool or double over float. This is because they want the code to be easier to read.

<h2>The “this” type</h2>

You may annotate with <em>this</em> when you return an instance of the class. Keep in mind that this could be an instance of a child class. This is a valid use of the this type:


{% highlight php %}
&lt;?hh
class Foo {
  private int $x = 0;
  public function setX(int $var): this {
    $this-&gt;x=$var;
    return $this;
  }
}
{% endhighlight %}


You may not use the <em>this</em> annotation in the following situations. This is because Foo could be extended and $this is not the same type for the subclass as it is for the base class.


{% highlight php %}
&lt;?hh
class Foo {
  //this is wrong
  public function newFoo(int $var): this {
    return new Foo();
  }
  //this is wrong
  public function newSelf(int $var): this {
    return new self();
  }
}
{% endhighlight %}

<h2>Do I really have to?</h2>

I said that Hack is statically typed. That is not true... It is <strong>gradually typed</strong>. This means that you may omit the type annotation if you like. But doing this the type checker will (of course) not be able to help you find type related bugs. I highly encourage you to always use type annotations.


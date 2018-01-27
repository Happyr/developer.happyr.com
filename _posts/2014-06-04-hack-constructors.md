---
title: Hack constructors
author: Tobias Nyholm
date: '2014-06-04 06:21:31 +0200'
categories:
- Hack
tags: []
---

Hack got a lot of features and improvements of different kinds. It comes with some syntactic sugar. The constructor is a subject for a syntactic sugar improvement.


The PHP-way of writing a class might look like this:
{% highlight php %}
&lt;?php


class Foobar {
  protected string $name;
  private int $count=0;


  public function __construct(string $name, int $count) {
    $this-&gt;name=$name;
    $this-&gt;cuont=$count;
    $this-&gt;init();
  }
  private function init() { }
}
{% endhighlight %}


The “problem” with this is that the spelling error on line 7 might get undetected. The Hack type checker might help you with this if all the supertypes are written in Hack. But in PHP, you might not find this bug until production.


With the new syntactic sugar in Hack you may write the same (without the spelling error) class variable and constructor like this:
{% highlight php %}
&lt;?hh


class Foobar {
  public function __construct(protected string $name, private int $count) {
    $this-&gt;init();
  }
  private function init() { }
}
{% endhighlight %}

<h2>Annotations</h2>

It is tempting to annotate constructors with void or this. Both make kind of sense since you are not returning anything, but then again you are getting an instance of the class. But how will it work when using this and class inheritance?
To avoid all confusion, don’t annotate a return type on the constructor. In fact, the type checker will throw an error if you do annotate the return type of a constructor.


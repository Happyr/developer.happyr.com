---

status: publish
published: true
title: Method declarations in Hack
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 544
wordpress_url: http://developer.happyr.com/?p=544
date: '2014-05-30 07:17:10 +0200'
date_gmt: '2014-05-30 05:17:10 +0200'
categories:
- Hack
tags: []
---

This post will go more in details of my <a title="Hack types" href="http://developer.happyr.com/hack-types">last post about types</a>. After reading that last post you know that a method declaration in Hack will look like this:


{% highlight php %}
//hack
class Foo {
  public function bar (int $var): string {
    return &quot;Number: &quot;.$var;
  }
}
{% endhighlight %}


I will dig in to some problems that isn’t obvious at the moment.

<h2>Inheritance</h2>

When Child extends Base the types will work as expected.


{% highlight php %}
//hack
class Base {}
class Child extends Base {}


class MyClass {
    public function valid1():Base
    {
        return new Child();
    }


    public function valid2():Child
    {
        return new Child();
    }


    public function notValid():Child
    {
        return new Base();
    }
}
{% endhighlight %}


When you are working with interfaces you need to be aware how you change the type.


{% highlight php %}
//hack
class Base {}
class Child extends Base {}
class GrandChild extends Child {}


interface FooInterface {
    public function getChild():Child;
}


//Valid: Foo returns GrandChild which is a subtype of Child
class Foo implements FooInterface {
    public function getChild():Child
    {
        return new GrandChild();
    }
}


//Invalid: Bar returns Base which is a supertype Child
class Bar implements FooInterface {
    public function getChild():Child
    {
        return new Base();
    }
}


{% endhighlight %}

<h2>Inheritance from PHP</h2>

When your Child class implements an interface from PHP. You need to be aware of what you are doing. You cant use Child::increase(int):int.  This is because Base::increase says: “I’ll handle inputs of any type” and Child::increase says: “I’ll handle just ints”. Since int is a narrower type this will be throw an error saying incompatible type. You should use <em>mixed</em>.


{% highlight php %}
//php
interface Base
{
    public function increase($var) {
        return $var + 1;
    }
}


//hack
require &quot;Base.php&quot;;
class Child extends Base {
     public function increase(mixed $var):int {
        return $var+2;
     }
}
$obj=new Child();
echo $obj-&gt;increase(2);
{% endhighlight %}


If Base is a PHP interface you will get an error since the interface rely on a wider type:

<blockquote>
Fatal error: Declaration of Child::increase() must be compatible with that of Base::increase() in /path/to/Child.hh on line 5
</blockquote>

<strong> </strong>

<h2>Methods with the same name</h2>

In Java you may have multiple methods with the same name but with different types of parameters or different parameter length. If the same applied in Hack you could write something like this:


{% highlight php %}
//hack
class Foobar {
     public function increase(int $var):int {
         return $var+2;
     }
     public function increase(string $var):string {
         return &quot;Many &quot;.$var;
     }
}
$obj=new Foobar();
echo $obj-&gt;increase(1).&quot;\n&quot;; //should output 3
echo $obj-&gt;increase(&quot;Apples&quot;).&quot;\n&quot;; //should output &quot;Many Apples&quot;
{% endhighlight %}


But the type checker will raise an error here:

<blockquote>
Foobar.hh:11:21,28: Name already bound
</blockquote>

This is a feature not available in Hack. Java's type checker is always running in “strict mode”. Since Hack may run in partial mode you may exclude the argument type annotation if you want to. That makes it impossible to know what method to invoke. You will find more about overloading methods in Hack this <a href="https://github.com/facebook/hhvm/issues/2532">Github issue</a>.

<h2>More about subtypes and supertypes</h2>

When you are changing the type you could return a subtype and change the arguments into a supertype. The later is not supported in Hack at the time of writing. (HHVM version 3.0.1)


{% highlight php %}
//hack
class Base {}
class Child extends Base {}
class GrandChild extends Child {}


interface FooInterface {
    public function getChild(Child $var): void;
}


//Invalid: Foo::getChild takes a GrandChild as argument. GrandChild is a subtype of Child
class Foo implements FooInterface {
    public function getChild(GrandChild $var): void
    {


    }
}


//Invalid (but should be valid): Bar::getChild takes a Base as argument. Base is a supertype of Child
class Bar implements FooInterface {
    public function getChild(Base $var): void
    {


    }
}


{% endhighlight %}


This will generate an error like:

<blockquote>
This is an object of type Child, It is incompatible with an object of type Base
</blockquote>

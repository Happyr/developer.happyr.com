---
title: Hack modes
author: Tobias Nyholm
date: '2014-06-03 06:18:16 +0200'
categories:
- Hack
---

When you are using Hack you are running in one of 3 different modes. The modes are telling the type checker how strict (or forgiving) it should be when checking the code. The modes are strict, partial and decl. In addition to this there modes, there is a declaration called unsafe that is used to disable the type checker.


All modes are declared just after the opening <em>&lt;?hh</em> tag. They are case sensitive and it must be a space between the slashes and the mode.


{% highlight php %}
&lt;?hh // mode
{% endhighlight %}

<h2>Strict</h2>

{% highlight php %}
&lt;?hh // strict
{% endhighlight %}


Strict mode is the one I prefer the most. With this mode you will be forced to use type annotations everywhere. You will get errors when you are trying to call non-Hack code. One other feature (or consequence) of using strict mode is that you can’t use PHP arrays. You have to work with Collections.


The reason why I think this is the preferred mode is that you learn to love the type checker. The type checker is a great feature of Hack and you should use it as much as you can. If you writing code that is using/inheriting/implementing PHP code there is no shame in using the partial mode.

<h2>Partial</h2>

{% highlight php %}
&lt;?hh // partial
{% endhighlight %}


The partial mode is the default mode in Hack. Since it is default you are actually encouraged to omit the mode and just use the <em>&lt;?hh</em>. Everything you read about Hack on this blog is true in partial mode if nothing else has been stated. You know from the <a href="http://developer.happyr.com/hack-types">post about types</a> that you may omit the type annotation if you want to.

<h2>Decl</h2>

{% highlight php %}
&lt;?hh // decl
{% endhighlight %}


The decl mode is the most forgiving mode. You use it to allow Hack code written in strict mode to call legacy code. That is the only reason I know of why decl mode exists. So you take your PHP file, change it to Hack in decl mode and you may use the file from a Hack file with strict mode on.


A consequence of the decl mode is that the type checker does not warn you about errors like this:


{% highlight php %}
&lt;?hh // decl
function foobar(): int {
  return &quot;Baz&quot;
}
{% endhighlight %}


Using decl mode is like putting unsafe declaration at the top of every function. (Which actually would be preferred over using decl mode.)

<h2>Unsafe</h2>

The unsafe declaration is a way to disable the type checker until the current block ends. It could be used in any Hack mode. But I have to give you a warning. When using unsafe you will get the type errors at runtime. So use this with caution.


{% highlight php %}
&lt;?hh
function foo(int $var): int {
  if ($var &gt; 5) {
    // UNSAFE
    return &quot;I am not checked by the type checker&quot;; // Covered by UNSAFE
  }
  return $var; // NOT covered by UNSAFE
}


function bar(int $var): int {
  if ($var &gt; 5) {
    return 6; // NOT covered by the UNSAFE
  }
  // UNSAFE
  if ($var &lt; 0) {
    return &quot;I am not checked by the type checker&quot;; // Covered by the UNSAFE
  }
  return true; // Covered by the UNSAFE
}
{% endhighlight %}


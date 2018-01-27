---
title: Hack interface overview
author: Tobias Nyholm
date: '2014-07-08 11:26:36 +0200'
categories:
- Hack
tags: []
redirect_from: "/hack-interface-overview/"
---
![Hack interfaces]({{ "/assets/images/HackInterfaces.jpg" | absolute_url }})

There are a number of predefined interfaces in Hack. This post is going to give you an overview about the interfaces that are a subtype of Traversable.

<h2>Traversable&lt;Tv&gt;</h2>

This is an abstract interface. Each an every class that implements this interface may be use with foreach()

<h2>KeyedTraversable&lt;Tk, Tv&gt;</h2>

This is an abstract interface. When implementing this you may use your object with a foreach with keyes. Like foreach($c as $k =&gt; $v)

<h2>IteratorAggregate&lt;Tv&gt;</h2>

This interface requires you to have a getIterator() function that will return an external iterator.

<h2>Iterable&lt;Tv&gt;</h2>

Interface to create an iterable object that will work with foreach()

<h2>KeyedIterable&lt;Tk, Tv&gt;</h2>

Interface to create an iterable object that will work with foreach() with keyes. Like foreach($c as $k =&gt; $v).

<h2>Indexish&lt;Tk, Tv&gt;</h2>

Allows you to use the bracket syntax.

<h2>Iterator&lt;Tv&gt;</h2>

Interface for external iterators or objects that can be iterated themselves internally.

<h2>KeyedIterator&lt;Tk, Tv&gt;</h2>

Same as Iterator but with keys.

<h2>Continuation&lt;Tv&gt;</h2>

This object is return from generators. The manual has an excellent <a href="http://docs.hhvm.com/manual/en/hack.continuations.php">example</a> of how to use Continuation.


---
title: Why collections?
author: Tobias Nyholm
date: '2014-06-10 16:52:59 +0200'
categories:
- Hack
---

Hack has introduced some new object to replace the PHP array. These collections are Vector, Set, Map, Pair and some more. If you are familiar with Java or C# you probably know the benefits of each of these collections already. You might be thinking:

<blockquote>
Why would you want to replace the PHP array? I use it for everything...
</blockquote>

That is the problem. PHP arrays are used everywhere and for everything. The keys might be ordered, unordered, strings, integers. A value might be a boolean, string or another array. It is impossible to predict the behavior of an array. PHP arrays are really “one size fits all”.

<h2>So, why collections?</h2>

Each collection object solves a specific problem. When you need to have a collection of unique values you should use a Set. If you need a mini data storage that you refer to with keys, you probably should use a Map.


Using collection objects have multiple benefits over the PHP array:

<ul>
<li>Better performance because the collection objects can be cached more aggressively.</li>
<li>PHP arrays are not objects they will be copied each time you pass them as a function parameter. Objects are passed by reference.</li>
<li>You may use the type checker with the collection objects</li>
<li>Collection object have function like filter(), map(), contains() etc to make them easier to work with.</li>
</ul>
<h2>Immutable instances</h2>

Most of the collection objects have an immutable version. Say that you have a Map that you have populated with some data. At some point in the Map object’s lifecycle you know that it should not be changed any more. Then you may create an ImmMap that does not allow any changes.


{% highlight php %}


function getFruits()
{
  $basket = Map {'apples'=&gt;4, 'oranges'=&gt;8, 'peaches'=&gt;2};
  $basket['bananas']=12;


  return $basket;
}


function askAStrangerToHoldBasket($basket)
{
  // ...
  return $basket;
}


$basket=getFruits();
$returnedBasket = askAStrangerToHoldBasket($basket-&gt;toImmMap());


//you can be sure that the stranger has not added or removed anything.


{% endhighlight %}


Immutable objects can be cached more aggressively because we know for sure that they will not change its state. They are also thread safe. But the most obvious reason is simplicity. They are easier to understand and predict.


If you design a card game, you could have each card being a immutable object since you know the cards are not going to change their values or suits… You will, however, never encounter a situation where you are required to use immutable objects. You could easily design the card game with mutable objects.

<h2>Vector</h2>

A Vector is an object with keys and values. The keys are integers starting from zero. Like in Java and C# you may access each element in the array on O(1) (constant time). You may also insert elements in the end of the vector at O(1).


{% highlight php %}


function foobar()
{
  $v = Vector {'Foo', 'Bar'};
  $v[]='Baz';


  echo $v[1].&quot;\n\n&quot;;


  $v-&gt;add('Biz');
  var_dump($v);
}


foobar();


{% endhighlight %}


Output:


{% highlight bash %}


Bar


object(HH\Vector)#1 (4) {
 [0]=&gt;
 string(3) &quot;Foo&quot;
 [1]=&gt;
 string(3) &quot;Bar&quot;
 [2]=&gt;
 string(3) &quot;Baz&quot;
 [3]=&gt;
 string(3) &quot;Biz&quot;
}


{% endhighlight %}


As you can see you may use square bracket syntax or the get/set functions. The square bracket syntax is preferable because it is faster and they are familiar to PHP developers.


Use a vector when you need any collection of elements where the index key is not important. This is the common replacement for a non-associative PHP array.


Be warned though. When removing elements from the Vector, the elements will get updated keys. If you try to access an element with a key that do not exist you will get an OutOfBoundsException.


{% highlight php %}


function foobar()
{
 $v = Vector {'Foo', 'Bar', 'Baz'};
 echo $v[2].&quot;\n\n&quot;;
 var_dump($v);


 echo &quot;\n -- \n&quot;;


 $v-&gt;removeKey(0);
 var_dump($v);


//Throws an OutOfBoundsException
 echo $v[2].&quot;\n&quot;;
}


foobar();


{% endhighlight %}


Outputs:


{% highlight bash %}


Baz


object(HH\Vector)#1 (3) {
[0]=&gt;
 string(3) &quot;Foo&quot;
[1]=&gt;
 string(3) &quot;Bar&quot;
[2]=&gt;
 string(3) &quot;Baz&quot;
}


--
 object(HH\Vector)#1 (2) {
 [0]=&gt;
 string(3) &quot;Bar&quot;
[1]=&gt;
 string(3) &quot;Baz&quot;
}


{% endhighlight %}

<h2>Map</h2>

A Map is like a Vector but you are in control of the keys. At the time of writing you may only use strings and integers as keys but you will eventually be able to use any object. Accessing an element in a Map is slower than in a Vector. Hack internal has to search for the key and then access the value. It has a time complexity of O(lg n).


{% highlight php %}
function foobar()
{
    $v = Map {'Foo'=&gt;'Good', 'Bar'=&gt;'Better'};
    $v['Biz'] = 'Okey';
    var_dump($v);
    echo &quot;\n\n&quot;;


    $v-&gt;removeKey('Foo');
    var_dump($v);


    //Throws an OutOfBoundsException
    //echo $v['Baz'].&quot;\n&quot;;
}


foobar();
{% endhighlight %}


Outputs:
{% highlight bash %}


object(HH\Map)#1 (3) {
[&quot;Foo&quot;]=&gt;
  string(4) &quot;Good&quot;
[&quot;Bar&quot;]=&gt;
  string(6) &quot;Better&quot;
[&quot;Biz&quot;]=&gt;
  string(4) &quot;Okey&quot;
}


object(HH\Map)#1 (2) {
[&quot;Bar&quot;]=&gt;
  string(6) &quot;Better&quot;
[&quot;Biz&quot;]=&gt;
  string(4) &quot;Okey&quot;
}


{% endhighlight %}

<h2>Set</h2>

A Set is a group of values without any keys. You may only store strings and integers more data types will be implemented later. Access in a Set has the time complexity of O(lg n) because Hack has to do a search for the element within the Set. You may create a Set from a Map or Vector by running the function toSet().


The special feature with Sets is that it can’t contain doublets. Each value is unique. If you try to assign the same value twice it will just be ignored.
 You might have used an array like this before:


{% highlight php %}
//php
function getFruits(array $stores)
{
    $fruits=array();
    foreach ($stores as $store) {
      foreach ($store-&gt;getFruits() as $fruit) {
        if (/* condition */) {
            $fruits[$fruit-&gt;getName()]=true;
        }
    }


    //return a list of unique fruit names that will meet the conditions
    return array_keys($fruits);
}
{% endhighlight %}


This is where you should use a Set.


{% highlight php %}
//hack
function getFruits(array $stores)
{
    $fruits=Set{};
    foreach ($stores as $store) {
      foreach ($store-&gt;getFruits() as $fruit) {
        if (/* condition */) {
            $fruits[]=$fruit-&gt;getName();
        }
    }


    //return a list of unique fruit names that will meet the conditions
    return $fruits;
}
{% endhighlight %}


When you are working with Sets you may use intersect.


{% highlight php %}function foobar()
{
    $foo=Set{'A', 'B', 'C'};
    $bar=Set{'B', 'D', 'A'};


    $baz=array_intersect($foo, $bar);
    var_dump($baz);
}


foobar();
{% endhighlight %}
Outputs:
{% highlight bash %}
array(2) {
    [&quot;A&quot;]=&gt;
  string(1) &quot;A&quot;
    [&quot;B&quot;]=&gt;
  string(1) &quot;B&quot;
}


{% endhighlight %}

<h2>Pair</h2>

A Par is like an immutable vector with 2 elements. I’ve not jet figured out any good use case for a Pair. In many cases it will be better to use a tuple or a shape.


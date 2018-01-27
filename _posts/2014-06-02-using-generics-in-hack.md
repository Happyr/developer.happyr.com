---
title: Using generics in Hack
author: Tobias Nyholm
date: '2014-06-02 06:32:06 +0200'
categories:
- Happyr
---

Hack has introduced some new collection objects that will be a replacement for the array(). But what if you feel that something is missing? What if you have a problem that is easiest solved with another type of object? This is where we use generics.


Say that you want to build a chess game. You may find it useful to have a class representing the board.


{% highlight php %}


class Matrix {
    private Map $matrix;
    public function __construct()
    {
        $this-&gt;matrix = new Map();
    }


    public function get(int $row, string $col): ?ChessPiece {
        if (!$this-&gt;matrix-&gt;containsKey($row)) {
            return null;
        }


        return $this-&gt;matrix-&gt;get($row)-&gt;get($col);
    }


    public function set(int $row, string $col, ChessPiece $value): void {
        if (!$this-&gt;matrix-&gt;containsKey($row)) {
            $this-&gt;matrix-&gt;set($row, new Map());
        }


        $this-&gt;matrix-&gt;get($row)-&gt;set($col, $value);
    }
}


{% endhighlight %}


This will probably work well for you. But when your manager ask you to build a Tic-tac-toe you have to rebuild your board like to be a Matrix[int][int]=Mark.


A better solution here is to use generics. You use generics as placeholders for types that are unknown at the time of writing. It is still statically typed because the types are defined when you are using an object of that class. Below you see an example of the matrix with generics.


{% highlight php %}
class Matrix&lt;Trow,Tcol,Tv&gt; {


    private Map&lt;Trow, Map&gt; $matrix;


    public function __construct()
    {
        $this-&gt;matrix = new Map();
    }


    public function get(Trow $row, Tcol $col): ?Tv {
        if (!$this-&gt;matrix-&gt;containsKey($row)) {
            return null;
        }


        return $this-&gt;matrix-&gt;get($row)-&gt;get($col);
    }


    public function set(Trow $row, Tcol $col, Tv $value): void {
        if (!$this-&gt;matrix-&gt;containsKey($row)) {
            $this-&gt;matrix-&gt;set($row, new Map());
        }


        $this-&gt;matrix-&gt;get($row)-&gt;set($col, $value);
    }
}


$matrix = new Matrix();
$matrix-&gt;set(1,2,'hello');
$matrix-&gt;set(2,3,'world');
echo $matrix-&gt;get(1,2);


{% endhighlight %}


It is important to know the difference between generic and variable. A generic could be any value but can not change after it got its type. See this example with Vector&lt;T&gt;


{% highlight php %}


function foobar() {
  $x = Vector {1, 2, 3, 4}; // T is associated with int
  $y = Vector {'a', 'b', 'c', 'd'}; // T is associated with string
}


foobar();


{% endhighlight %}


$x is of type Vector&lt;int&gt; and $y is of type Vector&lt;string&gt;. $x and $y are not of the same type.


It is good practice to name generic types with capital T in the beginning. You may use as many generics you want. Generic types can be used on both classes and functions.

<h2>Not supported features of generics</h2>

One documented unsupported of generics is to create a new instance of a generic type. This does not work.


{% highlight php %}


class Foo&lt;T&gt; {
	private ?T $data;


	public function __construct(?T $data) {
		$this-&gt;data=$data;
	}


	public function getNewOrExisting():T {
		if ($this-&gt;data===null) {
			return new T();
		}
		return $this-&gt;data;
	}
}


{% endhighlight %}

<h2>Conclusion</h2>

Generics are great when you want to create a wrapper object for something or when you are making your own data types. I am about to write a blog post about how the type checker works and how it works with generics. Stay updated.


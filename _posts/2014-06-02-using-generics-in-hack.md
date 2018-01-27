---

status: publish
published: true
title: Using generics in Hack
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 563
wordpress_url: http://developer.happyr.com/?p=563
date: '2014-06-02 06:32:06 +0200'
date_gmt: '2014-06-02 04:32:06 +0200'
categories:
- Happyr
tags: []
comments:
- id: 451
  author: romain
  author_email: kurt_le_punk922@hotmail.com
  author_url: http://noneyet
  date: '2014-06-15 21:02:43 +0200'
  date_gmt: '2014-06-15 19:02:43 +0200'
  content: "Hello. Thank you for this post. It was of great help to me. I first thought
    of using templates.. generic type just as in C++ or Java. But you just have to
    write a variable and it uses its type as the generic type... Not bad after all
    ! \r\nthank you again and good luck !"
- id: 589
  author: Josh Watzman
  author_email: jwatzman@fb.com
  author_url: ''
  date: '2014-07-01 19:05:11 +0200'
  date_gmt: '2014-07-01 17:05:11 +0200'
  content: In your second example, you still have an untyped Map -- I think you could
    make it even more type safe by typing it as "private Map&lt;Trow, Map&gt; $map"!
- id: 590
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2014-07-01 19:27:37 +0200'
  date_gmt: '2014-07-01 17:27:37 +0200'
  content: Thank you @Josh.
- id: 609
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2014-07-02 14:34:36 +0200'
  date_gmt: '2014-07-02 12:34:36 +0200'
  content: Thanks. I'm glad you like it.
- id: 108051
  author: Prabhu
  author_email: e75026h7k@gmail.com
  author_url: http://www.facebook.com/profile.php?id=100003469600338
  date: '2016-02-14 13:31:59 +0100'
  date_gmt: '2016-02-14 12:31:59 +0100'
  content: This is not strictly ovaloreding but similar.public class Loading { public
    static <A> A test() {      System.out.println("String");      return null;    }    public
    static <B> B test() {      System.out.println("Number");     return null;  }  public
    static void main(String[] args) {  Loading.test();  Loading.test(); }}
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


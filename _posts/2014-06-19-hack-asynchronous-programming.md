---
title: 'Hack: Asynchronous programming'
author: Tobias Nyholm
date: '2014-06-19 22:18:24 +0200'
categories:
- Happyr
---

Hack has introduced asynchronous programming. This is really great but it is not really documented at the moment. Not at all actually... I’ve done some experimenting and I will try to introduce you to the new concept.


There is two new keywords async and await. Async is used to declare a function as asynchronous. You need to declare the function with async if you want to run it in parallel with other functions. Every async function will return an Awaitable<T>.


{% highlight php %}


function foo(int $a): int {
    //This will return a Awaitable.
    $asyncCall = bar();


    //The bar function may not have finished executing
    if (!$asyncCall-&gt;isFinished()) {
        echo &quot;bar has not finished executing...\n&quot;;
    }


    //Wait for bar to finish and join the threads
    $bar=$asyncCall-&gt;join();


    return $bar+$a;
}


async function bar(): Awaitable&lt;int&gt; {
    echo &quot;bar started\n&quot;;


    //sleep for one sec. Make sure we sleep async
    //This could also be a remote API call...
    await SleepWaitHandle::create(1 * 1000000);


    echo &quot;bar is done\n&quot;;


    return 3;
}


echo &quot;Result: &quot;.foo(4);


//output
bar started
bar has not finished executing...
bar is done
Result: 7


{% endhighlight %}


Instead of the join function you may use the await keyword to suspend the execution of the async function waiting for the WaitHandle to complete.


There is also a series of object inheriting WaitHandle<T> which you may us. Below is an example where you stack up asynchronous calls in a GenVectorWaitHandle<T> and execute them all at once.


{% highlight php %}


function main() : void
{
    $info = Vector {};
    for ($i = 0; $i &lt; 5; $i++) {
        $info[] = genInfo($i);
    }


    echo '[main] Calling Async Function'.&quot;\n&quot;;
    $asyncCall = GenVectorWaitHandle::create($info);


    echo '[main] Doing whatever...'.&quot;\n&quot;;


    echo '[main] Time To Request Async Return Information'.&quot;\n&quot;;
    $output = $asyncCall-&gt;join();


    // Output Vector Data
    foreach ($output as $key =&gt; $value)
    {
        echo '['.$key.'] =&gt; '.$value.&quot;\n&quot;;
    }
}


async function genInfo(int $id): Awaitable&lt;String&gt; {
    echo '[genInfo] Generating '.$id.&quot;\n&quot;;


    $tmp = [];
    $tmp['id'] = $id;
    $tmp['start'] = date('H:i:s');


    //do some heavy working.
    await SleepWaitHandle::create(mt_rand(1000000,15*1000000));


    $tmp['end'] = date('H:i:s');


    echo &quot;[genInfo] Completed $id\n&quot;;


    return json_encode($tmp);
}


main();


/*
 * This is a modified example from http://kernelcurry.com/blog/asynchronous-hack/
 */


//Output
[genInfo] Generating 0
[genInfo] Generating 1
[genInfo] Generating 2
[genInfo] Generating 3
[genInfo] Generating 4
[main] Calling Async Function
[main] Doing whatever...
[main] Time To Request Async Return Information
[genInfo] Completed 0
[genInfo] Completed 3
[genInfo] Completed 2
[genInfo] Completed 1
[genInfo] Completed 4
[0] =&gt; {&quot;id&quot;:0,&quot;start&quot;:&quot;19:46:00&quot;,&quot;end&quot;:&quot;19:46:03&quot;}
[1] =&gt; {&quot;id&quot;:1,&quot;start&quot;:&quot;19:46:00&quot;,&quot;end&quot;:&quot;19:46:08&quot;}
[2] =&gt; {&quot;id&quot;:2,&quot;start&quot;:&quot;19:46:00&quot;,&quot;end&quot;:&quot;19:46:05&quot;}
[3] =&gt; {&quot;id&quot;:3,&quot;start&quot;:&quot;19:46:00&quot;,&quot;end&quot;:&quot;19:46:04&quot;}
[4] =&gt; {&quot;id&quot;:4,&quot;start&quot;:&quot;19:46:00&quot;,&quot;end&quot;:&quot;19:46:08&quot;}
{% endhighlight %}


Notice the last 5 lines in the output. The threads started at the very same time but completed at random order.


I think this feature need some more work before anyone should use it in production. We also need some more documentation on hacklang.org.


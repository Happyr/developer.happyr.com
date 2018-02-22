---
title: Testing the Symfony 4.1 router
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  teaser: images/posts/routelarge.jpg
  caption: "Photo credit: [**Flickr**](https://www.flickr.com/photos/rockys_photos/355216949/in/photolist-6enFxo-KGos4s-bziePy-91523N-TpGWrG-dvMSg3-6ZREpd-xozCa-3CVfrH-4r4Dz7-pkqvKY-5CL4FN-fHW3WW-8vvEYu-4KfNRw-6xwQuD-6xHB5q-8xo6sb-7dphxn-7KgWrr-342fEp-6xX2tf-6dBTjp-9nB8TA-8xk5pz-P61pjV-dNXAvC-5n8o7A-u7GL9y-6jzRhi-MGt7o1-7A7jLS-i4tC8C-4hDeBQ-3eX22G-dvGvj4-7A3gUD-5b6sLw-5yjxW2-8hxfmE-4cTxKp-Cq8sKM-tQg5e-7A7m1d-6ygJN3-avLA1p-9mQJiX-8ZtQtV-rpFoVZ-7A1DLZ)"
categories:
- Symfony
---
<b>How much faster is the new Symfony router in a real world application?</b>

<b>We ran some tests to find out.</b>

## An improved router

A few days ago we received the fantastic news that the Symfony router had been improved significantly. Of course we wanted to see how
the changes would affect our application, so we decided to test it against our own routes. 

If you somehow missed the news you can read the [official announcement](https://symfony.com/blog/new-in-symfony-4-1-fastest-php-router).

If you want to know how the new router works, read Nicolas Grekas [post](https://medium.com/@nicolas.grekas/making-symfonys-router-77-7x-faster-1-2-958e3754f0e1) where he dives more into detail about the new changes.  

## Our setup 
* PHP version: 7.1.13
* Static routes: 285
* Dynamic routes: 350

## The test process 
The process in itself is quite simple. Since the performance of the matcher is based on the amount of configured routes that we have, 
we want to test against the first route, some random routes, and the last route, to get a realistic test case. 
This was done for both the static and the dynamic routes. 

{% highlight php%}
class RouteTestCommand extends BaseCommand
{
    const NR_OF_MATCHES = 50000;
    protected static $defaultName = 'route:test';
    private $router;

    public function __construct(RouterInterface $router)
    {
        parent::__construct();
        $this->router = $router;
    }
    
    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('First static route: '.$this->matchRoutes(['/admin/statistics/index']));        
        // Add random/last static routes..

        $output->writeln('First dynamic route: '.$this->matchRoutes(['/admin/statistics/3aa63971-1f61-4b28-bd60-5bb1c9582c8a/show']));
        // Add random/last dynamic routes..
        
        $output->writeln('Not Found: '.$this->matchRoutes(['/foo/bar/baz']));
    }

    private function matchRoutes(array $routes): float
    {
        $startTime = microtime(true);
        foreach ($routes as $route) {
            for ($i = 0; $i < self::NR_OF_MATCHES; ++$i) {
                $this->router->match($route);
            }
        }
        $time = microtime(true) - $startTime;

        return number_format(1000 * $time / count($routes), 2);
    }
}
{% endhighlight %}

### Symfony 3.4
In Symfony 3.4, the matcher has to iterate through all/most of the configured routes and try to find the route for the given url. 
There are some optimizations like checking group of routes that have the same static prefix, and thereafter the routes in that subset. 
Trying to match a url towards the end of the list will result in an increased matching time, due to the number of comparisons 
being done before the route is found. 

That is why our tests for random route and last route take significantly more time to match than the first route. 
The routes that were not found will take the most amount of time to match, since all the configured routes must be iterated
before drawing the conclusion that our route does not exist. 

### Symfony 4.1
Instead of making separate comparisons or  `preg_match()` calls for each route, it combines all the regular expressions into a single regular expression.
This means that we only have to call `preg_match()` once, and that is the biggest factor for faster matching. 

If you'd like to read more about the regular expression optimization, check out [part 2 of Nicolas Grekas medium post](https://medium.com/p/making-symfony-router-lightning-fast-2-2-19281dcd245b). 

## Results
The table below shows how long it took to match the given routes 50.000 times. 

The __Diff__ column displays how much faster the new Symfony router was, compared to our current one (3.4).  

| Routes.               | 3.4    | 4.1 (7d29a4d) | Diff  |
| ----------------------|--------|---------------|-------|
| First static route    | 448 ms | 382 ms        | -17%  |
| Random static route   | 1621 ms| 474 ms        | -242% | 
| Last static route     | 1826 ms| 544 ms        | -234% | 
|                       |        |               |       |
| First dynamic route   | 746 ms | 527 ms        | -41%  |
| Random dynamic route  | 1454 ms| 531 ms        | -174% |
| Last dynamic route    | 2039 ms| 524 ms        | -289% |
|                       |        |               |       |
| Not Found             | 2112 ms| 522 ms        | -304% | 

As we can see, all of our routes were matched faster than previously. We especially see a gain in matching speed for 
the random, last, and when the route was not found.  

We are happy with these results, since the only thing we have to do once it is released is to 
run `composer update`.    

Remember that this is our results on our application. Your results will probably not be the same since the performance is highly dependent on the amount of routes
that your application has and on the tree structure of the routes. 

What results did you get?

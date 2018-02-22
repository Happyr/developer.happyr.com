---
title: Testing the new symfony router
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  teaser: images/posts/teaser2.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---
<b> How much faster is the new Symfony router? 
We ran some tests to find out.</b>

## An improved router

A few days ago we received the fantastic news that the Symfony router had been improved significantly, of course we wanted to see how
the changes would affect our application, so we decided to test it against our own routes. 

If you somehow missed the exciting news you can read this [post](https://medium.com/@nicolas.grekas/making-symfonys-router-77-7x-faster-1-2-958e3754f0e1) from Nicolas Grekas. 

## Our setup 
* PHP version: 7.1.13
* Routes: 635
* Static routes: 285
* Dynamic routes: 350

## Test 
During the test process we followed this [example](https://gist.github.com/nikic/9049180) made by Nikita Popov. 
The process in it self is quite simple. Since the router performance is based on the priority of the routes we have
to test a sample that contains routes with difference priorities. 
We chose to test against the first route, five random routes, and the last route. This was done for both the static
and the dynamic routes. 

{% highlight php%}
class RouteTestCommand extends BaseCommand
{
    const NR_OF_MATCHES = 50000;

    protected static $defaultName = 'route:test';

    /** @var Router */
    private $router;

    public function __construct(RouterInterface $router)
    {
        parent::__construct();
        $this->router = $router;
    }

    protected function configure()
    {
        $this->setDescription('Test the route speed');
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $output->writeln('First static route: '.$this->matchRoutes(['/admin/statistics/index']));        
        // Add random/last static routes..

        $output->writeln('First dynamic route: '.$this->matchRoutes(['/admin/statistics/3aa63971-1f61-4b28-bd60-5bb1c9582c8a/show']));
        // Add random/last dynamic routes..

        $output->writeln('Not Found: '.$this->matchRoutes(['/foo/bar/baz']));
    }

    private function matchRoutes(array $routes)
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

{% endhighlight %}

## Results
The table below shows how long each router took to match our given routes. 

The __Diff__ column displays how much faster the new Symfony router was, compared to our current one (3.4).  


| Routes(ms)            | 3.4    | 4.1   | Diff  |
| ----------------------|--------|-------|-------|
| First static route    | 448    | 382   | -17%  |
| Random static route   | 1621   | 474   | -242% | 
| Last static route     | 1826   | 544   | -234% | 
|                       |        |       |       |
| First dynamic route   | 746    | 527   | -41%  |
| Random dynamic route  | 1454   | 531   | -174% |
| Last dynamic route    | 2039   | 524   | -289% |
|                       |        |       |       |
| Not Found             | 2112   | 522   | -304% | 

As we can see all of our routes were matched faster than previously, some faster than others. We especially see a significant gain in matching speed for 
the random and the last routes. 

We are extremely happy with these results, since the only thing we have to do once it is released is to 
run `composer update`    

Remember that this is our results on our application. Your results will probably not be the same since the percentage difference is highly dependant on the amount of routes
that your application has. What results did you get?
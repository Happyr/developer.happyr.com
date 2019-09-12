---
title: Symfony HTTP Client and caching
author: Tobias Nyholm
date: '2019-09-12 14:35:00 +0200'
header: 
  image: "images/posts/containers.jpg"
  teaser: images/posts/containers_thumb.jpg
categories:
- Symfony
--- 

Symfony has a [HTTP client](https://symfony.com/doc/current/components/http_client.html) which differ from other clients
like Guzzle, Buzz or clients from the HTTPlug organization. Symfony's HTTP client is asynchronous by default. Using 
asynchronous anything is strange in PHP but there is no magic. The asynchronous part in any PHP HTTP client is achieved 
with help from cURL. 

Asynchronous by default means that we are only making the HTTP request when we actually need data from the response. This
allow us to start multiple requests and let cURL execute them in parallel. 

Consider a server that take one second to reply. 

{% highlight php %}
use Symfony\Component\HttpClient\CurlHttpClient;

$client = new CurlHttpClient();
$responses = [];

// Start clock
for ($i = 0; $i < 100; ++$i) {
    $responses[] = $client->request('GET', 'https://slow-server.com/foobar/'.$i);
}

foreach ($responses as $response) {
    $content = $response->getContent();
}
// Stop clock. Result: 1.5s
{% endhighlight %}

The code above will take just more than one second to execute all the HTTP requests and read the response. . 

The following code will make the requests in serial. 

{% highlight php %}
use Symfony\Component\HttpClient\CurlHttpClient;

$client = new CurlHttpClient();
$responses = [];

// Start clock
for ($i = 0; $i < 100; ++$i) {
    $response = $client->request('GET', 'https://slow-server.com/foobar/'.$i);
    $responses[] = $response->getContent();
}

// Stop clock. Result: 100+s
{% endhighlight %}

I hope this illustrate why Symfony's HTTP client is super cool. This is of course achievable in other clients but
Symfony has built their client with asynchronous in its foundation.  

## How about caching responses?

If some of these 100 requests have been executed previously, we could reduce the number of HTTP requests by using cache.
How could we leverage caching with these parallel requests?

The recommended way is to use the ``CachingHttpClient``, that will respect all cache headers sent by the server 
([See the documentation](https://symfony.com/doc/current/components/http_client.html#caching-requests-and-responses)). If 
you want more control or ignore the servers header we must use a more custom approach. This could be a good solution when you
are caching responses from a paid API (ie Google Translate). 

The general idea is to first look in the cache, if there is a cache miss, we start the request. Then we loop over all the
cache misses and fetch the response to store them in cache for later use. 

The full example looks like this. 

{% highlight php %}

$ids = [0, 1, 2, /*...*/ 98, 99];
$client = new CurlHttpClient();
$cache = new MyPsr6CachePool();

$responseUnions = [];
$responses = [];

/** @var CacheItemInterface[] $cacheItems */
$cacheItems = $cache->getItems(\array_map(function (int $id) {
    return 'my_id_'.$id;
}, $ids));

foreach ($cacheItems as $item) {
    $id = \mb_substr($item->getKey(), 6);
    if ($item->isHit()) {
        $responses[$id] = $item->get();
    } else {
        $responseUnions[$id] = [
            $item, // Store cache item in the union
            $client->request('GET', 'https://slow-server.com/foobar/'.$id)
        ];
    }
}

// If we did not have data in cache, fetch it.
foreach ($responseUnions as $id => [$item, $response]) {
    $responses[$id] = $response->getContent(false);
    $item->expiresAfter(3600);
    $item->set($responses[$id]);
    $cache->saveDeferred($item);
}

if (!empty($responseUnions)) {
    $cache->commit();
}

// Assert: $responses is now an array with all the response bodies. 
{% endhighlight %}

At first, this might seam to be a complex setup with a lot of things happening. Im not yet sure how to simplify this and
make the code more easy to read. . The code example works as a good template for future customizations. In a real world 
project you would probably use different cache lengths types depending on what response code you get. We would also need 
error handling and maybe response hydration. 

I hope this post gave some inspiration what you could use with the Symfony HTTP client. 

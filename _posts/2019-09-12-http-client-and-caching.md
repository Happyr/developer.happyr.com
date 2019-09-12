---
title: Symfony HTTP Client and caching
author: Tobias Nyholm
date: '2019-09-12 14:35:00 +0200'
header: 
  image: "images/posts/neo4j.jpg"
  teaser: images/posts/neo4j.jpg
categories:
- Symfony
--- 

Symfony has a [HTTP client](https://symfony.com/doc/current/components/http_client.html) which differ from other clients
like Guzzle, Buzz or clients from the HTTPlug organization. Symfony's HTTP client is asynchronous by default. Using 
asynchronous anything is strange in PHP but there is no magic. The asynchronous part in any PHP HTTP client is achieved 
with help from cURL. 

Asynchronous by default means that we are only making the request when we actually need data from the response. This
allow us to start multiple requests and let cURL do them in parallel. 

Consider a server that take one second to reply. 

{% highlight php %}
use Symfony\Component\HttpClient\CurlHttpClient;

// Start clock
$client = new CurlHttpClient();
$responses = [];
for ($i = 0; $i < 100; ++$i) {
    $responses[] = $client->request('GET', 'https://slow-server.com/foobar');
}

foreach ($responses as $response) {
    $content = $response->getContent();
}
// End clock: 1.5s
{% endhighlight %}

The above code will take just more than one second. 

The following code will make the requests in serial. 

{% highlight php %}
use Symfony\Component\HttpClient\CurlHttpClient;

// Start clock
$client = new CurlHttpClient();
$responses = [];
for ($i = 0; $i < 100; ++$i) {
    $response = $client->request('GET', 'https://slow-server.com/foobar/'.$i);
    $responses[] = $response->getContent();
}

// End clock: 100+s
{% endhighlight %}

I hope this illustrate why this is super cool. 

## How about caching?

If some of these 100 requests have been made previously, we could speed things up by using cache. How could we leverage 
caching with these parallel requests?

The general idea is to first look in the cache, if there is a cache miss, we start the request. Then we loop over all the
cache misses and fetch the response to store them in cache for later use. 

The full example looks like this. 

{% highlight php %}

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

At first, this might seam to be a complex setup with a lot of things happening. But one will get used to it. The 
code above works as a good template for future customizations. In a real projects you would probably use different resonse
types depending on what response you get. We would also need error handling and maybe response hydration. 

I hope this post gave some inspiration. 

---
title: Generate URLs in Symfony commands  
author: Tobias Nyholm
date: '2018-01-31 08:23:47 +0200'
header:
  image: "images/posts/little-man-hacker.jpg"
  teaser: images/posts/little-man-hacker.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---

It has been great many years since Symfony 2.0 was released. 2.0 did not have as many great features as Symfony 4.0, but 
it was flexible and you could still accomplish what ever you want. I've just found out one small thing that Symfony 4.0
does way better that 2.0. 

EDIT: This is not new to Symfony 4.0. It was a feature already in 2.7. 

Long long time ago I had the need for creating a Symfony command that sends emails to users given some criteria. So I wrote
my command and set up my cron job and everything was fine. Except the URLs. Whenever I rendered my HTML/Twig view as body 
in my email all the links got to localhost.   

{% highlight php %}
$url = $this->urlGenerator->generate('my_route', [], UrlGeneratorInterface::ABSOLUTE_URL);
echo $url; // "http://localhost/my-route"
{% endhighlight %}

The example above is in PHP but it will be the same if I try to render my email view in twig. 

The reason why I get "http://localhost" instead of my proper domain here is because the router has no context. The router
will automatically get a context when we have a Request but since this is CLI we will not have a request...

## A solution

What I did years ago and what I still use on many places is a custom service for rendering emails. You may inject a "RequestProvider"
into that service. The service uses that "RequestProvider" to get a context and locale and inject those to the RequestContext
and Translator. 

An example of the default request provider looked like this: 

{% highlight php %}
class RequestProvider implements RequestProviderInterface
{
    const DEFAULT_LOCALE = 'sv';

    /**
     * @var string siteUrl
     */
    protected $siteHost;

    /**
     * @var string siteSchema
     */
    protected $siteSchema;
    
    /**
     * @var EntityManagerInterface
     */
    private $em;

    public function __construct(EntityManagerInterface $em, $siteHost, $siteSchema)
    {
        $this->em = $em;
        $this->siteHost = $siteHost;
        $this->siteSchema = $siteSchema;
    }

    public function getRequest($email, $tmplData)
    {
        $request = new Request();
        $request->headers->set('host', $this->siteHost);
        $request->headers->set('schema', $this->siteSchema);

        $request->setLocale($this->getLocale($email, $tmplData));

        return $request;
    }

    protected function getLocale($email, $tmplData)
    {
        /** @var User $user */
        if ($user = $this->em->getRepository(User::class)->findOneBy(['email' => $email])) {
            $user->getLocale();
        }

        if (isset($tmplData['_original_locale'])) {
            return $tmplData['_original_locale'];
        }

        return self::DEFAULT_LOCALE;
    }
}
{% endhighlight %}

This works and it has been working well. But it is a bit complicated. 

## A much better solution

As I said before, Symfony 4 is great. You do not need to do all that complicated stuff. You may just need to add a few lines
of configuration. (From [symfony docs](http://symfony.com/doc/current/console/request_context.html).)

{% highlight yaml %}
# config/services.yaml
parameters:
    router.request_context.host: example.org
    router.request_context.scheme: https
    router.request_context.base_url: my/path
{% endhighlight %}

And about the locale, whenever I got the user I want to send an email to, I change the locale in the translator. 

{% highlight php %}
// Get the user I want to email
$user = $this->em->getRepository(User::class)->find(4711);
$this->translator->setLocale($user->getLocale() ?? $this->defaultLocale);
{% endhighlight %}

I really think I overdid my first implementation and I'm real happy this better solution exist. 
---
title: User as a service in Symfony2
author: Tobias Nyholm
date: '2015-04-21 14:54:47 +0200'
categories:
- Symfony2
---

To get the current logged in user in Symfony2 is kind of complicated. You have to do a lot for such simple task. First you have to get the TokenStorage to retrieve the token. The token may or may not exist. If the token exist you can get an user object or a string 'anon.'. It all looks like this:


{% highlight php %}
public function getUser()
{
    if (null === $token = $this-&gt;container-&gt;get('security.token_storage')-&gt;getToken()) {
        // no authentication information is available
        return;
    }


    if (!is_object($user = $token-&gt;getUser())) {
        // e.g. anonymous authentication
        return;
    }


    return $user;
}


{% endhighlight %}


The function above is provided by the FramworkBundle base controller. But what if you want to use the current user in a service? You have to inject the <em>security.token_storage</em> and do this yourself. Wouldn't it be great if we could do something like this in our service declaration:


{% highlight javascript %}


services:
  my_servce:
    class: Acme\Service\MyService
    arguments: [@current_user]


{% endhighlight %}


And then have the constructor of MyService like...


{% highlight php %}
public function __construct(User $user = null) {}
{% endhighlight %}

<h2>Use factories</h2>

We know that we can use a <a href="http://symfony.com/doc/current/components/dependency_injection/factories.html">factory to create a service</a>. What if we can use a factory to fetch the current user for us?


Lets create a factory class and register the <em>current_user</em> service.


{% highlight php %}
class CurrentUserFactory
{
    private $tokenStorage;


    public function __construct(TokenStorageInterface $tokenStorage)
    {
        $this-&gt;tokenStorage = $tokenStorage;
    }


    /**
     * Get the logged in user or null.
     *
     * @return User
     */
    public function getUser()
    {
        if (null === $token = $this-&gt;tokenStorage-&gt;getToken()) {
            return;
        }


        if (!is_object($user = $token-&gt;getUser())) {
            return;
        }


        return $user;
    }
}
{% endhighlight %}
{% highlight sql %}
services:
  current_user.factory:
    class: Acme\UserBundle\Factory\CurrentUserFactory
    arguments: [@security.token_storage]
  current_user:
    class:   Acme\UserBundle\Entity\User
    factory: [&quot;@current_user.factory&quot;, getUser]


{% endhighlight %}


This works and it is excellent. We only encounter a problem when we consider the <a href="http://symfony.com/doc/current/cookbook/service_container/scopes.html">dependency injection scopes</a> (which we always should). By default we register services in the <em>container</em> scope. That means that the services are created once and lives in memory until after kernel.terminate event. So if we got the following chain of events:

<ol>
<li>The user submits the login form</li>
<li>We instantiate the current_user service (which is null)</li>
<li>We authenticate the user and updates the TokenStorage with a new token</li>
<li>We try to use the current_user service</li>
</ol>

The current_user service will still be <strong>null</strong> in step 4. So let's change the scope of <em>current_user</em> to <em>prototype</em>. Now the service will be created every time it is used. This is exactly what we want! But when you declare <em>current_user</em> in scope <em>prototype</em> you will get this nasty exception:


![]({{ "/assets/images/scope_prototype_exception.png" | absolute_url }})

You may remember back in Symfony 2.3 when you tried to inject the <em>@request</em> into a service, then you got a similar exception. So how did Symfony solve this it? They introduced the <a href="http://symfony.com/blog/new-in-symfony-2-4-the-request-stack">RequestStack in 2.4</a>. This is just a service that you could push and pop Requests on. Yes, like a normal stack.

<h2>My recommended solution</h2>

To avoid the problems highlighted you should not use the user object as a service. The main reason is because it is not a service... It is a model. But what you do want to do is making it easier for yourself to retrieve a user. I suggest you create a CurrentUserProvider that has a function getUser. Then inject that CurrentUserProvider into your services. The implementation of CurrentUserProvider is very similar to the CurrentUserFactory showed above.


I have made a PR for a new <em>security.current_user_provider</em> service that hopefully will be merged into 2.8. Follow it here: <a href="https://github.com/symfony/symfony/pull/14407">https://github.com/symfony/symfony/pull/14407</a>


<b>Edit 2015-05-05:</b> When discussing this post with friends in the community and on the PR, I've been told that it is bad design if you need to inject the user into a service. You should pass the user object as an parameter from your controller. I will try to avoid the CurrentUserProvider when I write new features.


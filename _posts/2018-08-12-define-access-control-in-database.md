---
title: Define Symfony access control rules in a database
author: Tobias Nyholm
date: '2018-08-12 22:23:47 +0200'
header:
  image: "images/posts/little-man-hacker.jpg"
  teaser: images/posts/little-man-hacker.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---

I was recently at a PHP conference in Odessa where I met many great developers. 
One of them asked me a question, that the answer was not obvious. His use case
was that he wanted to use Symfonys Access Control configuration to restrict access 
in his application. But he also wanted to configure the rules dynamically. 

Since all the configuration in Symfony is cached with the container for performance
reasons, we could obviously not allow a use a database to somehow "print" new configuration.
We need to do something smarter. 

## Using Symfony voters

Symfony is using voters to decide access to URL and resources. There are many
voters that come with the [security component](https://github.com/symfony/symfony/tree/4.1/src/Symfony/Component/Security/Core/Authorization/Voter) 
that are using the configuration in `security.access_control` to decide if the request
should be granted or not. 

What we want to do is to create a new voter that access the database. How you create
a Voter is covered by the [Symfony documentation](https://symfony.com/doc/current/security/voters.html).   

{% highlight php %}
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\VoterInterface;

class MyDynamicAccessVoter implements VoterInterface
{
    private $em;
    
    public function __construct(EntityMangerInterface $em)
    {
        $this->em = $em;
    }
    
    public function vote(TokenInterface $token, $subject, array $attributes)
    {
        if (!$subject instanceof Request) {
            return self::ACCESS_ABSTAIN;
        }

        $uri = $subject->getUri();
        $roles = $token->getRoles();
        
        // TODO implement your logic here.
        //$this->em->getRepository(...)
        
        if (/* Has access */) {
            return self::ACCESS_GRANTED;
        }
        
        if (/* No access */) {
            return self::ACCESS_DENIED;
        }
        
        // Vote "abstain" if we do not know
        return self::ACCESS_ABSTAIN;
    }
}
{% endhighlight %}

This class should be registered in the service container. If you are using
Symfony 4 with the standard `services.yaml` it will be registered automatically. 
If you need to register it manually, remember to add the `security.voter` tag. 
{% highlight yaml %}
services:
    App\Voter\MyDynamicAccessVoter:
        arguments: ['@doctrine.orm.entity_manager']
        tags:
          - 'security.voter'
{% endhighlight %}

You also need to tell Symfony that you are using a access control: 

{% highlight yaml %}
security:
    access_control:
         - { path: ^/admin }
{% endhighlight %}

The `MyDynamicAccessVoter` will now be executed for every URL that starts with 
"/admin". You could of course change it to "/" to execute it on all requests. 

{% highlight yaml %}
security:
    access_control:
         - { path: ^/ }
{% endhighlight %}

Eventhough this solution is not obvious, it shows how powerful and flexible 
Symfony's security component is. 

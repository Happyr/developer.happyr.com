---
title: Using UUID with Symfony ParamConverter
author: Tobias Nyholm
date: '2018-08-12 22:23:47 +0200'
header:
  image: "images/posts/abstract0.jpg"
  teaser: images/posts/abstract0.jpg
  caption: "Photo credit: [**Pedro Ribeiro Simões**](https://www.flickr.com/photos/pedrosimoes7/)"
categories:
- Symfony
---

Symfony's ParamConverters are really helpful. I use them as often as I can. Instead of
writing this peace of code on every controller:

{% highlight php %}
/**
 * @Route("/{id}/show")
 */
public function show($id)
{
    $article = $this->em->getRepository(Article::class)->find($id);
    if (null === $article) {
        throw $this->createNotFoundException();    
    }
    
    // ... 
}
{% endhighlight %}

... I could just write: 

{% highlight php %}
/**
 * @Route("/{id}/show")
 */
public function show(Article $article)
{
    // ... 
}
{% endhighlight %}

Of course, this assumes that I've [installed and configured](https://symfony.com/doc/current/bundles/SensioFrameworkExtraBundle/index.html) 
the bundle correctly.  

## Enable UUID support

If you use UUID as your primary key that the above examples works great. But if 
you are not for some reason you need to do some more configuration. Say that your
entities looks like this: 

{% highlight php %}
class Article
{
    /**
     * @var int
     *
     * @ORM\Column(name="id", type="integer")
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @var string
     * @var \Ramsey\Uuid\Uuid
     *
     * @ORM\Column(type="uuid", unique=true)
     */
    private $uuid;
   // ...
}
{% endhighlight %}

You want to create controller with routes like this:

{% highlight php %}
class FooController
{
    /**
     * @Route("/{uuid}/show", requirements={"uuid": "%uuid_regex%"})
     */
    public function bar(Article $user) { /* ... */ }


    /**
     * @Route("/{uuid}/comments/{user_uuid}/show", requirements={"uuid": "%uuid_regex%", "user_uuid": "%uuid_regex%"})
     */
    public function baz(Article $article, User $user) { /* ... */ }
}
{% endhighlight %}

To achieve this we need to configure our own ParamConverter that supports UUID. 
We can use the same `DoctrineParamConverter` class but with some different config. 
This is supported since SensioFrameworkExtraBundle version 5.2.1.

{% highlight yaml %}
app.param_converter.uuid: 
    class: Sensio\Bundle\FrameworkExtraBundle\Request\ParamConverter\DoctrineParamConverter
    public: false
    arguments: 
        - '@doctrine'
        - '@sensio_framework_extra.converter.doctrine.orm.expression_language'
        - 
            repository_method: 'findOneByUuid'
            id: ['uuid', '%s_uuid']
{% endhighlight %}

Or the same thing for XML:

{% highlight xml %}
<service id="app.param_converter.uuid" class="Sensio\Bundle\FrameworkExtraBundle\Request\ParamConverter\DoctrineParamConverter" public="false">
    <tag name="request.param_converter" converter="doctrine.orm.uuid" />
    <argument type="service" id="doctrine" on-invalid="ignore" />
    <argument type="service" id="sensio_framework_extra.converter.doctrine.orm.expression_language" on-invalid="null" />
    <argument type="collection">
        <argument key="repository_method">findOneByUuid</argument>
        <argument key="id" type="collection">
            <argument>uuid</argument>
            <argument>%s_uuid</argument>
        </argument>
    </argument>
</service>
{% endhighlight %}

## Note

The ParamConverter belongs to [SensioFrameworkExtraBundle](https://github.com/sensiolabs/SensioFrameworkExtraBundle)
so if you want to use then, check their installation documentation. 


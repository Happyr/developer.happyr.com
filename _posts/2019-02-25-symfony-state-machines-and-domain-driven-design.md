---
title: Symfony State Machines and Domain Driven Design
author: Tobias Nyholm
date: '2019-02-25 22:23:47 +0200'
header:
  image: "images/posts/traffic-light.jpg"
  teaser: images/posts/traffic-light.jpg
  caption: "Photo credit: [**dalioPhoto**](https://www.flickr.com/photos/marcdalio/)"
categories:
- Symfony
---

It has been just over 2 years since Symfony released their Workflow component. 
I was of course thrilled by the news and started to work on multiple PRs to make
the component support state machines. It was a really cool experience working with 
the Symfony community and discussing everything from Petri nets to minor code 
optimizations. 

I often get questions how to work with the Workflow component and questions like
"what should I do in this really specific scenario". One question I do get a lot
is how one should work with the component when doing Domain Driven Design.  

Since I'm no expert in DDD I imminently put myself on very deep water writing 
this blog post. But here are my thoughts. 

In Domain Driven Design the model (or entity) should be responsible for itself.  
Only it should have the knowledge how to move from one state to another. The problem
with the workflow component is that you are defining the workflow's configuration
in a Yaml file. See example from [Symfony documentation](https://symfony.com/doc/current/workflow/usage.html#creating-a-workflow).

There is a way you can work around this though with some help from [service factories](https://symfony.com/doc/current/service_container/factories.html). 

Here is an example of a model. It is a blog post that could have a few different states: 
"draft", "published" and "trashed". I've created a static method on my model that 
will return a WorkflowDefinition. 

{% highlight php %}
namespace App\Entity;

class BlogPost {
  private $myState;
  // getters and setters

  public static function getWorkflowDefinition()
  {
        $definitionBuilder = new DefinitionBuilder();
        $definition = $definitionBuilder
            ->addPlaces(['draft', 'published', 'trashed'])
            ->addTransition(new Transition('publish', 'draft', 'published'))
            ->addTransition(new Transition('trash', 'published', 'trashed'))
            ->build();
            
        return $definition;
  }
}
{% endhighlight %}

I then create my service factory class. I've chosen to make this a "Workflow" factory
but you could easily modify this to be a "State Machine" factory.  

{% highlight php %}
namespace App\Workflow;

use Symfony\Component\EventDispatcher\EventDispatcherInterface;
use Symfony\Component\Workflow\MarkingStore\MultipleStateMarkingStore;
use Symfony\Component\Workflow\Validator\WorkflowValidator;
use Symfony\Component\Workflow\Workflow;

class WorkflowFactory
{
    private $eventDispatcher;

    public function __construct(EventDispatcherInterface $eventDispatcher)
    {
        $this->eventDispatcher = $eventDispatcher;
    }

    public function create(callable $fetcher, $name)
    {
        $definition = $fetcher();
        (new WorkflowValidator())->validate($definition, $name);
        $marking = new MultipleStateMarkingStore('myState');

        return new Workflow($definition, $marking, $this->eventDispatcher, $name);
    }
}
{% endhighlight %}

The final piece of the puzzle is to glue everything together and create your service
definition. Note that our WorkflowFactory is generic and could be reused for other 
workflows, not just the BlogPost.

{% highlight yaml %}
  App\Workflow\WorkflowFactory:
    arguments: ['@event_dispatcher']

  workflow.blog_post:
    class: Symfony\Component\Workflow\Workflow
    factory: ['@App\Workflow\WorkflowFactory', 'create']
    arguments: [['App\Entity\BlogPost', 'getWorkflowDefinition'], 'blog_post']
  
  # Other workflows can use the same factory
  workflow.acme:
    class: Symfony\Component\Workflow\Workflow
    factory: ['@App\Workflow\WorkflowFactory', 'create']
    arguments: [['App\Entity\Acme', 'getWorkflowDefinition'], 'acme']
{% endhighlight %}

I hope this small post has given you an idea how to be successful with the workflow
component when doing Domain Driven Design. 

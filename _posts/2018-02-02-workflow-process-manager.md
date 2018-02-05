---
title: Manage your workflows efficiently 
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  teaser: images/posts/teaser2.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---

This post will cover how to use a ProcessManger class together with Symfony's workflow component.
The ProcessManager is responsible for mapping states with controller routes. This makes it easy for you to know which
controller action that is next in line and ready to be executed. The ProcessManager class looks like this: 

{% highlight php%}
class ProcessManager
{
    /**
     * A map for states to routes.
     *
     * @var array
     */
    private $routes;

    /**
     * @var Workflow
     */
    private $workflow;

    /**
     * @param Workflow $workflow
     * @param array    $routes   state=>route
     */
    public function __construct(Workflow $workflow, array $routes)
    {
        $this->workflow = $workflow;
        $this->routes = $routes;
    }

    /**
     * @param mixed $subject
     *
     * @throws \LogicException
     *
     * @return string
     */
    public function getCurrentRoute($subject)
    {
        $markings = $this->workflow->getMarking($subject)->getPlaces();

        // It is a single marking store.
        reset($markings);
        $marking = key($markings);

        if (isset($this->routes[$marking])) {
            return $this->routes[$marking];
        }

        throw new \LogicException(sprintf('The route for place "%s" was not found in ProcessManager for "%s"', $marking, $this->workflow->getName()));
    }
}

{% endhighlight %}

A workflow is very similar to a state machine, containing places and transitions, where places refer to "states".
To read more about configuring and setting up your own workflow, please see the Symfony documentation: [here](https://symfony.com/doc/current/components/workflow.html)

While redesigning some of our step-by-step applications in our system we decided to start using Symfony's workflow component as a state machine.
This made much sense, since it would benefit us by having one state for each step of the process.  


The first step in your controller would be to retrieve your state machine from the service container:
{% highlight php%}
$stateMachine = $this->get('workflow.application_create');
{% endhighlight %}

In the next step, check if the transition you want to apply is available for your object, based on its current state.
If it is, apply it and let Doctrine update the database:
{% highlight php%}

if ($stateMachine->can($application, 'pay')) {
    $stateMachine->apply($application, 'pay');

    $em = $this->getEntityManager();
    $em->persist($application);
    $em->flush();
}

{% endhighlight %}

Now comes the interesting part:
After applying the new state, you would also want to redirect the user to the next step. This is where the ProcessManager comes in.
Before you can start using the ProcessManager you have to declare it in services.yml, since it is a service.
{% highlight php%}
services:

  application.process.manager:
    class: App\Workflow\ProcessManager
    arguments:
      - "@workflow.application_create"
      -
        create_account: 'register'
        payment: 'pay'
        done: 'done'

{% endhighlight %}

The argument sent to the ProcessManager is the specific workflow that we want our ProcessManager to handle.

<p>As we can see in the declaration above, each state has a route name assigned to it. For an example: 
The state <mark>create_account</mark> is mapped to the <mark>'register'</mark> route name.  

</p>

Each route name refers to a specific
controller action, and this is handled by [annotations](http://symfony.com/doc/current/bundles/SensioFrameworkExtraBundle/annotations/routing.html).
<p>Therefore by using the ProcessManager in your controller you can safely say that you're are executing the correct controller,
and that the user is redirected to the correct page. </p>

{% highlight php %}

// Retrieve our ProcessManager and get the current route for our application object
$route = $this->get('application.process.manager')
    ->getCurrentRoute($application);

return $this->redirect($this->generateUrl($route, ['uuid' => $application->getUuid()]));

{% endhighlight%}

A final example of a controller action would look like this: 

{% highlight php %}

public function paymentAction(LightWeightApplication $application, Request $request)
{
    
    // Create your form here 

    if ($request->isMethod('POST')) {
        $stateMachine = $this->get('workflow.application_create');
        if ($stateMachine->can($application, 'payment')) {
            $stateMachine->apply($application, 'payment');

            $em = $this->getEntityManager();
            $em->persist($application);
            $em->flush();
        }

        $route = $this->get('application.process.manager')
            ->getCurrentRoute($application);

        return $this->redirect($this->generateUrl($route, ['uuid' => $application->getUuid()]));
    }

    return [
        'application' => $application,
    ];
}

{% endhighlight %}
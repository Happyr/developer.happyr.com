---
title: Using a process manager with the workflow component
author: Georgos Landin Xanthopoulos
date: '2018-02-02 15:06:47 +0200'
header:
  image: "images/posts/little-man-hacker.jpg"
  teaser: images/posts/little-man-hacker.jpg
  caption: "Photo credit: [**The Preiser Project**](https://www.flickr.com/photos/thepreiserproject/)"
categories:
- Symfony
---

A workflow is very similar to a state machine, containing places and transitions, where places refer to "states".
To read more about configuring and setting up your own workflow, please see the Symfony documentation:[here](https://symfony.com/doc/current/components/workflow.html)

While redesigning some of our step-by-step applications in our system we decided to start using Symfony's workflow component as a state machine.
This made much sense, since it would benefit us by having one state for each step of the process.

Here is an example of how the entire flow could look like:
![]images/posts/workflow.jpg

Below you can see a typical controller action that is responsible for one of the steps in the application process:

{% highlight php %}

public function infoAction(LightWeightApplication $application, Request $request)
{
    if (null !== $response = $this->verifyRequest($application, 'surveillance_info')) {
        return $response;
    }

    if ($request->isMethod('POST')) {
        $stateMachine = $this->get('workflow.application_create');
        if ($stateMachine->can($application, 'surveillance_info_completed')) {
            $stateMachine->apply($application, 'surveillance_info_completed');

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

The first thing we always do in our action methods is to check that the current state of the object matches the expected state of our action.
We wouldn't want to proceed otherwise.
{% highlight php%}

if (null !== $response = $this->verifyRequest($application, 'test')) {
    return $response;
}
{% endhighlight%}

The next step is very straight forward. We retrieve our state machine from the service container:
{% highlight php%}
$stateMachine = $this->get('workflow.application_create');
{% endhighlight %}

In the next step, we check if the transition we want to do is available for our object, based on its current state.
If it is, apply it and let Doctrine update our database:
{% highlight php%}

if ($stateMachine->can($application, 'invitation_accepted')) {
    $stateMachine->apply($application, 'invitation_accepted');

    $em = $this->getEntityManager();
    $em->persist($application);
    $em->flush();
}

{% endhighlight %}

Now comes the interesting part:
After applying the new state, we also want to redirect the user to the next step. This is where the ProcessManager comes in.
The ProcessManager is responsible for mapping states with controller routes. This makes it easy for us to know which
controller action that is next in line and ready to be executed.

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

Since our ProcessManager is a service we declare it in our services.yml:
{% highlight php%}
services:

  application.process.manager:
    class: App\Workflow\ProcessManager
    arguments:
      - "@workflow.application_create"
      -
        surveillance_info: 'user_surveillance_info'
        start: 'user_surveillance_start'
        job_interests: 'user_surveillance_job_interests'
        test_information: 'user_surveillance_test_information'
        test: 'user_surveillance_before_test'
        surveillance_done: 'user_surveillance_test_done'

{% endhighlight %}

The argument sent to the ProcessManager is the specific workflow that we want our ProcessManager to handle.

As we can see in the declaration above, each state has a route name assigned to it. Each route name refers to a specific
controller action, and this is handled by [annotations](http://symfony.com/doc/current/bundles/SensioFrameworkExtraBundle/annotations/routing.html).
Therefore by calling getCurrentRoute() in our actions we can safely say that we are executing the correct controller,
and that the user is redirected to the correct page.

{% highlight php %}

// Retrieve our ProcessManager and get the current route for our application object
$route = $this->get('application.process.manager')
    ->getCurrentRoute($application);

return $this->redirect($this->generateUrl($route, ['uuid' => $application->getUuid()]));

{% endhighlight%}








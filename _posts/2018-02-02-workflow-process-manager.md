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

<b>This post will cover how to use a ProcessManager class together with Symfony's workflow component. </b> 

## The Process Manager
The ProcessManager is responsible for mapping states with controller routes. This makes it easy for you to know which
controller action that is next in line and ready to be executed.

The ProcessManager class looks like this: 

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

To read more about configuring and setting up your own workflow, please see the [Symfony documentation](https://symfony.com/doc/current/components/workflow.html)

## Solution 
The first thing to do in your controller would be to retrieve your state machine from the service container:
{% highlight php%}
$stateMachine = $this->get('workflow.checkout');
{% endhighlight %}

Next, check if the transition you want to apply is available for your object, based on its current state.
If it is, apply it and let Doctrine update the database:
{% highlight php%}

if ($stateMachine->can($order, 'pay')) {
    $stateMachine->apply($order, 'pay');

    $em = $this->getEntityManager();
    $em->persist($order);
    $em->flush();
}

{% endhighlight %}

Now comes the interesting part:
After applying the new state, you would also want to redirect the user to the next step. This is where the __ProcessManager__ comes in.

Before you can start using the __ProcessManager__ you have to declare it in services.yml:
{% highlight php%}
services:

  checkout.process.manager:
    class: App\Workflow\ProcessManager
    arguments:
      - "@workflow.checkout"
      -
        create_account: 'account_register'
        products: 'products_add'
        payment: 'order_payment'
        done: 'order_done'

{% endhighlight %}

The argument sent to the __ProcessManager__ is the specific workflow that we want our __ProcessManager__ to handle.

As we can see in the declaration above, each state has a route name assigned to it. For an example: 
The state <mark>create_account</mark> is mapped to the <mark>'account_register'</mark> route name.  

Each route name refers to a specific
controller action, and this is handled by [annotations](http://symfony.com/doc/current/bundles/SensioFrameworkExtraBundle/annotations/routing.html).
<p>With the state-to-route mapping, the ProcessManager will always make sure that the users is redirected to the correct controller. </p>

{% highlight php %}

// Retrieve our ProcessManager and get the current route for our order object
$route = $this->get('checkout.process.manager')
    ->getCurrentRoute($order);

return $this->redirect($this->generateUrl($route, ['uuid' => $order->getUuid()]));

{% endhighlight%}

A final example of a controller action would look like this: 

{% highlight php %}

public function paymentAction(Checkout $order, Request $request)
{
    // Create your form here 
    $form = // .. 

    if ($form->isSubmitted() && $form->isValid()) {
        $stateMachine = $this->get('workflow.checkout');
        if ($stateMachine->can($order, 'payment')) {
            $stateMachine->apply($order, 'payment');

            $em = $this->getEntityManager();
            $em->persist($order);
            $em->flush();
        }

        $route = $this->get('checkout.process.manager')
            ->getCurrentRoute($order);

        return $this->redirect($this->generateUrl($route, ['uuid' => $order->getUuid()]));
    }

    return [
        'order' => $order,
    ];
}

{% endhighlight %}
---
title: Symfony 4.4 preload on AWS Lambda
author: Tobias Nyholm
date: '2019-12-23 17:30:00 +0200'
header:
  image: "images/posts/sns-retry.png"
  teaser: images/posts/sns-retry.png
categories:
- Bref
- Symfony
---

PHP 7.4 preloading sounds amazing and the Symfony community have does some great 
things so I can easily enable preloading in my application. So this is a very short
post with some numbers and statistics. 

Results will of course vary between applications. 

## My application

I have a very small application that I deploy on AWS Lambda using Bref. Running code
on Lambda is not quick, but it is okey. The server got 1792Mb of memory.  

The application has about 150 files in the src folder with a total of 5300 lines of 
code. The specific route I was testing with does 8 database calls, renders 7 templates
and has 5 template blocks. 

The controller just fetches data from the database and displays it: 

{% highlight php %}
final class RecruitmentController extends AuthenticatedAbstractController
{
    /**
     * @Route("/recruitments", name="recruitment_index", methods={"GET"})
     */
    public function index(RecruitmentRepository $repository): Response
    {
        $customer = $this->getLoggedCustomer();
        $recruitments = $repository->findAllForCustomer($customer);

        return $this->render('manage/recruitment/index.html.twig', [
            'recruitments' => $recruitments,
        ]);
    }
}
{% endhighlight %}

## Sampling data

So I started by running a baseline by refreshing the page every 2 seconds. 
When I had enough data I redeployed the application with adding the following
line to php.ini

{% highlight ini %}
opcache.preload=/var/task/var/cache/prod/srcApp_KernelProdContainer.preload.php

{% endhighlight %}

Then I started to refresh the page again to sample data a second time. 

## The data

### Baseline

Samples: 357 requests
Average response time: 65,13 ms

### Preloading

Samples: 357 requests
Average response time: 65,13 ms


## Conclusion


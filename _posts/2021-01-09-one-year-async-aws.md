---
title: One year of AsyncAws
author: Tobias Nyholm
date: '2021-01-09 11:18:00 +0200'
header:
  image: images/posts/woman-forest.jpg
  teaser: images/posts/woman-forest-thumb.jpg
  caption: "Photo credit: [**Chanh Duong**](https://unsplash.com/@lemoniron)"
---

It has been one year since Jérémy Derussé and I started to work on a new API client
for AWS. At the time we could never have imagined how popular it would be. In this
first year we had over 1.5 million downloads over all packages. The core package
alone has been downloaded more than 500.000 times.

The story behind the project is simple. We wanted an AWS client that was fundamentally
different from the official one. The big features are all described on [async-aws.com](https://async-aws.com/),
but in short: we use small packages, readable code, great memory management and
asynchronous HTTP responses. This makes the AsyncAws client really easy to work
with and it is fast! Sure, it is still HTTP…. but as quick as you can make it.

Both Jérémy and I maintain a number of open source packages, but AsyncAws is special
for both of us. One really interesting aspect is that most of the code is generated
from a specification. The developers at AWS have specified all their API requests
and responses. From these specifications we can generate PHP code that will generate
the proper XML/JSON requests and response parsers. This will ensure that the API
clients are always correct. When AWS updates their API specification, a bot will
automatically make a PR to update the AsyncAws client code.

We spent a lot of time on the code generator. We are really happy with how it turned
out, but it is not yet complete. There might be some APIs that have some endpoints
that are not yet supported. That is only natural because at the beginning we only
focused on a handful of popular AWS services. Then we received feature requests
and now we support around 20 AWS services. That is just 1% of the available AWS
APIs but it covers 90% of all the usages. So the code generator only has support
for features that are used, not “all” features. While working very closely with
the API specifications we've noticed quite a few inconsistencies. They are mostly
about formats and naming etc. But it is a fresh reminder that even a great company
like AWS also is not perfect and has a lot of legacy code.

The AsyncAws organisation has three main parts; the code generator, the authentication
system, and the connection with the Symfony HTTP client. Then of course there are
some docker images, bots and the website. The authentication system is considered
feature complete. All the 6 official ways to authenticate are supported and working.

The integration with Symfony HTTP client is what gives us the power and flexibility
of asynchronous HTTP. I am also very happy to say that the relationship is mutually
beneficial. The work on the Symfony's RetryableHttpClient for example, was started
by Jérémy in a PR for AsyncAws. This integration might be complex when you dig in
the details, but it is simple to understand on a high level. It should make sure
that all requests are always sent, throw exceptions and log things when appropriate
and do as little of the slow HTTP as possible.

## Is it difficult to migrate from the official SDK?

You probably already have an application that is using the official AWS SDK. It is
surprisingly easy to migrate to AsyncAws. All services have mostly the same namespace,
the ApiClient's methods are the same, they have the same option names and values.
This is thanks to the API specification mentioned earlier. If you ever are in doubt,
you can open up the AsyncAws client and read the code. You also get help from static
analyzer tools like Psalm and PHPStan.

If your framework is using a Flysystem integration for example, the only thing you
need to do is to update your config. You will also discover that it is easier to
write unit tests with AsyncAws. No more mocking PSR-7 responses!


## What about the future?

There are no big plans for the future. All current clients are stable. The bots will
automatically update them when the API changes. We will add a few minor features
and keep help adding more AWS clients when someone makes a request for it. There
are already integrations for Laravel, Symfony, Flysystem, Monolog etc. There are
even people reporting that they use AsyncAws for AWS S3 compatible services from
Digital Ocean and other providers.

I hope you try it out and enjoy using it as much as we have enjoyed creating it.

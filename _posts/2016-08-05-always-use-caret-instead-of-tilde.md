---

title: Always use caret instead of tilde
author: Tobias Nyholm
date: '2016-08-05 08:23:47 +0200'
categories:
- Happyr
---

I've been noticing people having trouble understanding the differences between the caret ("^") and the tilde ("~") operator
in the composer.json file. Composer's documentation is great but a bit short, that is why I write this blog post.


Composer assumes that all packages are using Semver. It is an easy version schema that consists of a major.minor.patch version. A major version breaks backwards compatibility, minor version introduces new features and patch versions are bug fixes.


The tilde operator is best explained by example. Notice that it actually makes a difference if you specify the patch version or not.

{% highlight bash %}
~1.0   means >=1.0.0 <2.0.0 same as 1.*
~1.0.0 means >=1.0.0 <1.1.0 same as 1.0.*
{% endhighlight %}

The caret operator follows Semver.

{% highlight bash %}
^1.0   means >=1.0.0 <2.0.0 same as 1.*
^1.0.0 means >=1.0.0 <2.0.0 same as 1.*
{% endhighlight %}

What is little less known is that Semver behaves differently on versions under 1.0.0. Before the project has a stable release, minor versions are allowed to break backwards compatibility. This is dangerous when you use the tilde operator since you may get update that breaks backwards compatibility.

{% highlight bash %}
// This is dangerous as updates might break BC
~0.2 means >=0.2.0 <1.0.0
// This is fine
~1.2 means >=1.2.0 <2.0.0
{% endhighlight %}

A workaround for this could be that you always specify versions under 1.0.0 with an asterisk ("*").

{% highlight bash %}
0.2.* means >=0.2.0 <0.3.0
{% endhighlight %}

A better approach is to continue to use the caret operator as it still respects Semver even on versions under 1.0.0.

{% highlight bash %}
^0.2 means >=0.2.0 <0.3.0
{% endhighlight %}

In conclusion: One should always use caret ("^") over tilde ("~") because caret is always respecting Semver.


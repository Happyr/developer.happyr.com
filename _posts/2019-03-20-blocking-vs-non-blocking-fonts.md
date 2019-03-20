---
title: Blocking vs non-blocking fonts
author: Tobias Nyholm
date: '2019-03-20 10:38:47 +0200'
header:
  image: "images/posts/non-blocking-fonts.png"
  teaser: images/posts/non-blocking-fonts.png
categories:
- Web
---

Just the other day I found out about the [font-display: swap;](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display)
css directive. It basically allows you to configure the browser to load the webpage
first and then add the correct fonts when they are loaded. 

Note that you can only add `font-display: swap;` on `@font-face`. That is an issue 
because we are using Google fonts. So I downloaded the CSS file from Google fonts
and stored it locally. Then I added  `font-display: swap;` on all `@font-face` like: 


{% highlight css %}
@font-face {
  font-family: 'Roboto';
  font-style: normal;
  font-display: swap;
  font-weight: 500;
  src: local('Roboto Medium'), local('Roboto-Medium'), url(https://fonts.gstatic.com/s/roboto/v18/KFOlCnqEu92Fr1MmEU9fBBc4AMP6lQ.woff2) format('woff2');
  unicode-range: U+0000-00FF, U+0131, U+0152-0153, U+02BB-02BC, U+02C6, U+02DA, U+02DC, U+2000-206F, U+2074, U+20AC, U+2122, U+2191, U+2193, U+2212, U+2215, U+FEFF, U+FFFD;
}
{% endhighlight %}

I also did the same for our Material Icons. 

The result is actually shocking. Here is
a video illustrating the differences.  I have throttled the network to "regular 3G".
I reload the page as soon as the video starts. The non-blocking version shows the 
content almost instantly. This is a way better user experience.  

<iframe width="560" height="315" src="https://www.youtube.com/embed/T62gzrxvTz8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe> 
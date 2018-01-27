---
title: Symfony bundle using Hack
author: Tobias Nyholm
date: '2014-05-12 09:47:52 +0200'
categories:
- Symfony2
- HHVM
- Hack
tags: []
---

I've been working with the <a href="http://www.symfony.se">Symfony.se</a> website the last couple of weeks. In a discussion on Github we argued how a nice excerpt should look like. We decided that the excerpt twig filter should take a HTML string as input and make it shorter without breaking the HTML. It should also remove tables and convert headings.


Since Symfony.se runs on HHVM I decided to write the Twig filter in Hack. The bundle is not optimized or optimal, it is written to demonstrate some cool Hack features. You will find the bundle <a href="http://developer.happyr.com/symfony2-bundles/excerpt-bundle">here</a> and the cool hack features is found in <a href="https://github.com/HappyR/ExcerptBundle/blob/master/Service/HackExcerpt.php" target="_blank">this file</a>. I wanted to make sure you could run the symfony.se website on a normal PHP installation as well. So I added two excerpt services. One written in PHP and one written in Hack. In the dependency injection configuration I made a check if HACK_VERSION is defined to decide which service to load.


If you want to try the Hack bundle with the symfony.se project I recommend you reading my post about <a title="Try Hack and HHVM" href="http://developer.happyr.com/try-hack-and-hhvm">how to install HHVM and Hack</a>.

<h2>What to look at</h2>

When you are reading the <a href="https://github.com/HappyR/ExcerptBundle/blob/master/Service/HackExcerpt.php">HackExcerpt class</a> pay extra attention to:

<ul>
<li>the first rows declaring the Heading shape</li>
<li>the constructor</li>
<li>the method declaration of <em>getDefaults</em></li>
<li>the <em>preg_replace_callback</em> function in <em>convertHeadings</em></li>
</ul>

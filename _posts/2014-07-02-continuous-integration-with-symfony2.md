---

status: publish
published: true
title: Continuous integration with Symfony2
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 203
wordpress_url: http://developer.happyr.com/?p=203
date: '2014-07-02 13:50:36 +0200'
date_gmt: '2014-07-02 11:50:36 +0200'
categories:
- Developing
- Symfony2
tags: []
---

<strong>There are many great articles that tells you why you should use continuous integration i.e. the one by <a href="http://martinfowler.com/articles/continuousIntegration.html">Martin Fowler</a> and the book This is Lean by Niklas Modig and Pär Ahlström. This article describes our workflow and what tools we use to achieve continuous integration and extreme programming. We are doing most of our developing in Symfony2 and we are using tools like Jenkins, Sonar, Ant and a bunch of homemade services and scripts. </strong>


We use <a href="http://jenkins-ci.org">Jenkins</a> as a continuous integration server. Jenkins is a great platform where you can add on other services that may run analysis, tests, generate reports etc. Whenever we do a push to our Git server, Jenkins will start to "build" the project. We have defined an <a href="http://ant.apache.org">Ant</a> target to specify what to do when the project is being built. The instruction file for Ant is called build.xml and it is basically a list of tasks. (You will find a reference to our build.xml at the bottom of this page.) At each build we tell Ant to:

<ol>
<li>Purge the cache</li>
<li>Create some folders, like app/build/logs</li>
<li>Run composer update</li>
<li>Run PHPUnit</li>
</ol>

We have instructed PHPUnit to generate both test reports and code cover reports. After all the above Ant tasks have succeeded, Jenkins will send the code to quality analysis.

<h2>Code analysis with Sonar</h2>

Sonar is a great peace of software from <a href="http://www.sonarqube.org">Sonarsource</a>. It is a platform for code analysis. Sonar supports a lot of common programming languages. I've installed the PHP package which contains HP Depend, PHP Mess Detector, PHP CodeSniffer. Together with the Sonar core and a some nice quality profiles it will help us find bugs and make the code uniform. We've made some strict rules that will fire alarms if the code contains more than 5 major violations or if the minor violations has increased with more than 15 since last week. When a alarm is fired Jenkins will mark the build as failed.


Except from code style Sonar will also measure complexity and find logical bugs. Sonar once told me I had to rewrite a function because it was over 100 possible execution ways through that function...


The first couple of weeks with Sonar was pain. Each commit we did had a lot of violations and we thought it was just slowing us down. But soon we learned how to write code with no violations and it really made our code more uniform and easy to read and maintain.


Another great code analysis software for PHP is the <a href="https://insight.sensiolabs.com/">SensioLabs insight</a>. Sonar and SensioLabs insight should both be used as they complement each other. SensioLabs insight detects much more sophisticated errors and security vulnerabilities.


Download our <a href="http://developer.happyr.com/wp-content/uploads/2013/07/Sonar_profile_Symfony2_php.xml_.zip">Symfony2 Sonar quality profile</a>.

<h2>When build fails</h2>

When a build fails (i.e. when the PHPUnit or the analysis fails) Jenkins will stop the execution flow and send an email to whoever broke the build. The code must not proceed to the development or production servers.


I've read some examples of companies who have implemented some sort of monitoring of the builds. You may see build status in Jenkins but it would be a lot cooler to have a red lava lamp that gets turned on when the code in the master branch does not pass the tests or quality profile. If the build is broken for 15 minutes or more the lava lamp bubbles are starting to move. =)

<h2>Distribution to test and production</h2>

If the build passes both the test suite and the quality profile it is time to deploy the project to a development server. This is done at each successful build. You may run more manual tests and/or QA tests on this server. It is also a good place to let you manager try the new features.


Jenkins should also be in charge of deployment to the QA server and production server. I prefer to start those deployment myself by clicking on a button in Jenkins but I know of developers that would like to deploy to production when pushing to the "production" git branch.


Our deployment is a Ant target. Ant will:

<ol>
<li>Do a git pull</li>
<li>Put the site in maintenance mode</li>
<li>Run composer install</li>
<li>Migrate the database and clear APC cache</li>
<li>Install new bower dependencies</li>
<li>Run Assetic dump</li>
<li>Clear and warm up the cache</li>
<li>Remove the maintenance mode</li>
</ol>

Since Jenkins is running on a different server than many of our application I tell Jenkins that a deployment is to start a shell script. The shell script uses ssh to get to the production server and then start the Ant task from there. It is possible to set up Jenkins to communicate with other servers but I prefer to keep things simple.


Download our Ant <a href="http://developer.happyr.com/wp-content/uploads/2014/06/build.xml_.zip">build.xml</a>.

<h2>Choose your tools</h2>

Which tools and services I use is not too important. What you should learn form this article is the workflow, the concept and that automation is key. If you are a .NET developer you might file that TeamCity is a much better choice over Jenkins or if you feel like using Capistrano instead of Ant that is fine. The concept is the same and you will achieve the same results no matter the name of the tool.


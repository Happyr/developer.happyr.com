---
title: How Happyr work with Symfony translations
author: Tobias Nyholm
date: '2015-08-24 12:50:59 +0200'
categories:
- Happyr
- Symfony2
tags: []
---

You probably follow the Symfony best practices when you develop web applications. That is great but there is no much information about translation. The <a href="http://symfony.com/doc/current/best_practices/i18n.html">official documentation</a> tell you about file format, file location and something about translations keys. But then what?


Some very talented members of the community have created <a href="http://jmsyst.com/bundles/JMSTranslationBundle">JMSTranslationBundle</a> which works if you (the developer) are translating everything yourself. But when you have requirement to support more than three languages and you need to hire professional translators you cannot use JMSTranslationBundle anymore. Neither can you pass around XLIFF files back and forth.


This article is going to argument about pros and cons when doing translations in a Symfony application. I will also describe what our workflow looks like.

<h2>What does not work</h2>
<h3>Using files</h3>

Symfony recommends using XLIFF as a file format for translation. That is the open standard for localisation files since 2002. There are tools like <a href="https://michelf.ca/software/counterparts-lite/" rel="nofollow">Counterparts Lite</a> (Mac) or <a href="http://sourceforge.net/projects/eviltrans/files/Transolution/Transolution%200.4b5%20%28Genesis%29/" rel="nofollow">Transolution</a> (Win)  that you can download and use to create new XLIFF files more languages. There are also online tools like <a href="https://poeditor.com/" rel="nofollow">POEditor</a> and <a href="http://xliff.brightec.co.uk/" rel="nofollow">Brightec Online XLIFF Editor</a> that you can use. They all suffer from the same issue. They are static.


You need to create your XLIFF file from Symfony, send it to the translator that works for a day? Two days? A week? When they are done they send the file back and you add it to the application again. How do you handle new strings that are created in the application when the XLIFF file is at the translators? Do you resend a new file? It will be a constant sending files back and forth. We don’t do that with code why should we do that with translation files?
One could argue that you could put all the translation files into a new git repository. This idea has two flaws. First you need to teach the translators how to git and secondly, you don’t want to handle merge conflicts in translation files.

<h3>Using JMSTranslationBundle</h3>

The workflow with <a href="https://github.com/schmittjoh/JMSTranslationBundle">JMSTranslationBundle</a> is that you scan you source code for translation strings and then dump them into file. You can then use a web UI to edit those translations. The issue here is: where should you host that web UI? You can have it on your local machine and it will be fine for you but what about the translators? They cannot and should not set up a development environment on their own.


You could allow translators to access the web UI on a staging server, then you have to write a script that commits the changes back in git, resolves any conflict and pushes it to the repository. You do also have to figure out when the script needs to be executed. Is it every hour? On every file change? Etc.


This method could actually work, but it is a lot of trouble. You need to maintain that new host and you need to make sure your continuous integration server pushes changes to it at every build. But there is no good way to deal with branches. Say that you develop a nice big feature in a separate branch then you cannot start translating the new strings until they are merged into master and get pushed to the translation host.

<h3>Using a database</h3>

There are some bundles that allow you to store the translations in a database. A bundle with this feature that I have previuosly worked with is <a href="https://github.com/lexik/LexikTranslationBundle">LexikTranslationBundle</a>. Like the JMSTranslationBundle does LexikTranslationBundle also provide a web UI for a developer to add and modify translations and the suggested workflow is very similar. The <a href="https://github.com/liip/LiipTranslationBundle">LiipTranslationBundle</a> have taken the web UI one step further by allowing you to inline edit your translations direct in your application. This is great because you do always know where your translations are displayed.


The drawbacks of these bundles is the same as before. You still have to import/export files from the translators or find a good way to export/import a database dump from staging, QA and production.

<h2>What does work</h2>

What you want to do is to use a third party solution for your translations. This will give your translators a platform to do their work without being dependent on the developers. The developers may do all the crazy branching, merging and rebasing they want and it does not bother the translation team. Most third party solutions are also feature rich and have a nice user interface. That is something you don’t really bother with if you’re creating your own translation solution.


There are plenty translation SaaS to choose from. We use <a href="https://localise.biz/"><b>Loco</b> by Tim Withlock</a> because it is feature rich and has a well written and expressive API. Here are a list of candidates I’ve considered.

<ul>
<li><a href="https://localise.biz/" rel="nofollow">Loco</a></li>
<li><a href="http://www.transifex.com/" rel="nofollow">Transiflex</a></li>
<li><a href="https://crowdin.com/" rel="nofollow">Crowdin</a></li>
<li><a href="http://openl10n.io/" rel="nofollow">OpenLocalization</a> (Open source and hosted by yourself)</li>
<li><a href="https://poeditor.com/" rel="nofollow">POEditor</a></li>
<li><a href="https://phraseapp.com/" rel="nofollow">PhraseApp</a></li>
<li><a href="http://www.oneskyapp.com/" rel="nofollow">OneSky</a></li>
<li><a href="https://www.getlocalization.com/" rel="nofollow">GetLocalization</a></li>
<li><a href="https://webtranslateit.com/en" rel="nofollow">WebTranslateIt</a></li>
<li><a href="https://www.localeapp.com/" rel="nofollow">Locale</a></li>
<li><a href="https://weblate.org/en/" rel="nofollow">Weblate</a></li>
</ul>

No matter which one you choose you should be able to download translations files, create new translations and maybe flag fuzzy translations by API calls.

<h2>Should translation files be in git?</h2>

There are both pros and cons having your translation files in a version repository. The drawbacks are, as stated before, that you need to push and pull and bother with merge conflicts. Cliff Odijk's (<a href="https://github.com/cmodijk">@cmodijk</a>) has made an argument that if you sometimes release an older branch you do not want to have the latest translations shipped with that branch. Because translations may have changed and “the button below” might not exist any more.


In our development at Happyr we do always release the latest updates so we don’t need to include the translation files in the version control. They are perfectly hosted in Loco and are downloaded at every deploy.

<h2>Give the translators a context</h2>

It is really hard translating an application, especially when you don’t know if the source string belongs to an error message, button label or what the surrounding texts are about. One important piece to solve this problem is to realise that translation is not something you do once and then you are done. It is the same with development. You do not develop your application and then fire all your developers. It is a process. You will not get it right the first time and you will have bugs to correct.


To ease with the process, make sure that your translation platform got features for flagging translations as incorrect or fuzzy. You should also be able to leave comments to describe the strings and give the translators some more information.

<h3>Use good translation keys</h3>

The Symfony best practises states that you should use <i>keys</i> instead of <i>strings</i>. They also state that:


“Keys should always describe their purpose and not their location. For example, if a form has a field with the label "Username", then a nice key would be label.username, not edit_form.label.username.”


I do not agree with this at 100%. I believe this is only true for very short or reusable strings. Like ‘label.username’, ‘login’, ‘show_all’, ‘paginator.next’ etc. The keys you are using for a paragraph or for some instruction texts should give the translator context and information about where the text is actually used. Like ‘acme.user.settings.subscription.paragraph0’ or ‘acme.advert.candidate_overview.sidebar.stats.info’.


There is a good blog post written by Damien Alexandre at <a href="http://jolicode.com/blog/translation-workflow-with-symfony2">jolicode.com</a> where he arguments for enforcing a key standard. By using a key standard you can give the translators a lot of context. Here is some example of keys we are using:

<table>
<tbody>
<tr>
<th>Key</th>
<th>Description</th>
</tr>
<tr>
<td><b>label.</b>foo</td>
<td>For all my labels</td>
</tr>
<tr>
<td><b>flash.</b>foo</td>
<td>For all my flash messages</td>
</tr>
<tr>
<td><b>error.</b>foo</td>
<td>For error messages</td>
</tr>
<tr>
<td><b>help.</b>foo</td>
<td>For help text the we use with our forms</td>
</tr>
<tr>
<td>foo<b>.heading</b></td>
<td>For a heading</td>
</tr>
<tr>
<td>foo<b>.paragraph</b>0</td>
<td>For a paragraph of text</td>
</tr>
<tr>
<td><b>_</b>foo</td>
<td>I start translation with an underscore if the translated string should start with a lowercase letter</td>
</tr>
<tr>
<td><b>foo</b></td>
<td>For any common strings like “Show all”, “Next”, “Yes” etc</td>
</tr>
<tr>
<td><b>vendor.bundle.controller.action.</b>foo</td>
<td>For any non-reusable translation</td>
</tr>
</tbody>
</table>

You should also make sure to use different domains. We use domains for <b>mail</b>, <b>messages</b>, <b>navigation</b> and <b>validators</b>. We also have a domain for <b>admin</b> but it does not need to be translated by a translator.

<h2>Our translation workflow</h2>

So, with all of the above in mind, how do we work with translations at Happyr? The very first thing we do is to download all the available translations from Loco with a console command. Then we start developing new features. Whenever the Symfony WebProfiler toolbar tells us we’ve got missing translations we access the translation page in the WebProfiler.


<img src="https://raw.githubusercontent.com/Happyr/TranslationBundle/master/src/Resources/doc/images/toolbar-example.png" alt="Image of the toolbar with missing translations" />


From here we can choose to send these missing translations to Loco by a click of a button.


<img src="https://raw.githubusercontent.com/Happyr/TranslationBundle/master/src/Resources/doc/images/missing-translation-example.gif" alt="Image where you send missing translations" width="400px" />


If you want to treat your translators well (which you should) you could add a translation string for one language. Because it could be difficult for a translator to know what the content of ‘acme.demo.show.paragraph0’ should be. This could also be done by the WebProfiler page. You can also edit a translation or flag it as “fuzzy”. A flagged translation gets highlighted for the translators and will tell them to rework it.


<img src="https://raw.githubusercontent.com/Happyr/TranslationBundle/master/src/Resources/doc/images/edit-flag-sync-example.gif" alt="Example of edit, sync and flag" width="400px" />


When you want to fetch the latest work from the translators you may run a sync command to synchronize all translations. You could also synchronize them one by one via the WebProfiler page.


Since we are not committing our translation files into a version repository we need to download the latest translations available from Loco at every deploy. This was easily integrated into our deployment script.


If you like our way of doing translations you should have a look at <a href="https://github.com/Happyr/TranslationBundle">HappyrTranslationBundle</a> where we’ve integrated all the features showed above.

<h2>Notes</h2>
<ul>
<li>We have a different Loco project for each translation domain. The only reason for that is that I may want to have the same key translated to different strings for different domains. If you don’t have this request you may use one project and separate domains with Loco filters.</li>
<li>The HappyrLocoBundle is missing a feature to add translations to Loco that you don’t see in the WebProfiler toolbar.</li>
<li>You are allowed to change the translation key in Loco, it might be an upcoming feature in the Bundle as well.</li>
</ul>

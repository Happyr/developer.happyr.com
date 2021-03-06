---
title: Do not worry about code style fixes
author: Tobias Nyholm
date: '2015-07-31 16:14:03 +0200'
categories:
- Happyr
- Developing
---

Earlier this year I installed Fabien's <a href="https://packagist.org/packages/friendsofphp/php-cs-fixer">php-cs-fixer</a> on my continuous integration server. So at each build I checked if there were any code style errors, if so I failed the build. Easy as pie and our source code were always nice and clean. But this new feature changed my workflow. My new workflow looks like this:

<ul>
<li>Write code</li>
<li>Commit & push code</li>
<li>[some time later]</li>
<li>Received a build error report</li>
<li>I ran php-cs-fixer</li>
<li>Commit & Push</li>
</ul>

I always seams to forget the to run php-cs-fixer before I push my code. The solution is call <a href="https://git-scm.com/book/it/v2/Customizing-Git-Git-Hooks">git hooks</a>. I can tell git that "For this repository, run this script before each commit". I edit the file .git/hooks/pre-commit and add the following line:


{% highlight bash %}


#!/bin/sh


git diff --name-only --diff-filter=MA HEAD~1 | grep -e "\.php\|\.sh\|\.twig\|\.js\|\.css\|\.md$" | xargs -L1 php-cs-fixer --fixers=-phpdoc_to_comment --level=symfony -q fix


{% endhighlight %}


This require you to have php-cs-fixer script in your path. What the script does it it takes all files modified since last commit, filter out php, sh, twig, js, css and md files and gives the rest to php-cs-fixer. I have turned off the 'phpdoc_to_comment' fix.


Every time I commit in this repository the php-cs-fixer will make sure I follow the coding standards.


You can do the same globally, but when you fetch a third party repository and do a quick fix you don't want to change all the coding standards in the files you edit.


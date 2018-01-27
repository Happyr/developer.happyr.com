---

status: publish
published: true
title: Choose owning side in OneToOne relation
author: Tobias Nyholm
author_login: Tobias Nyholm
author_email: tobias@happyr.com
wordpress_id: 623
wordpress_url: http://developer.happyr.com/?p=623
date: '2014-06-26 12:16:41 +0200'
date_gmt: '2014-06-26 10:16:41 +0200'
categories:
- Symfony2
- Performance
- Doctrine
tags: []
comments:
- id: 1107
  author: Brian
  author_email: briankosborne+happyr@gmail.com
  author_url: ''
  date: '2014-07-16 19:21:15 +0200'
  date_gmt: '2014-07-16 17:21:15 +0200'
  content: But then you could have lots of NULL fields on the owning side (user),
    unless every User has a Resume. It doesn't seem natural to have the FK on the
    User to me. One way to get around the extra queries is to manually JOIN the Resume
    record in when you are loading a User. That way Doctrine won't have to fetch it
    individually.
- id: 1112
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2014-07-17 13:52:28 +0200'
  date_gmt: '2014-07-17 11:52:28 +0200'
  content: "Yes, It might be null on the $user->getResume(). But that is okey. \r\n\r\nI
    have looked into the solution of manually JOIN the Resume entity when I'm loading
    the user. It is quite some work with the User entity because you have to dig into
    the Security component. But anyhow, I don't want to spend any memory on extra
    entities when I know I'm not going to use them. \r\n\r\nSo instead of working
    around Doctrine, I choose to reconfigure my entities in a way that Doctrine has
    intended."
- id: 1157
  author: Iltar van der Berg
  author_email: kjarli@gmail.com
  author_url: ''
  date: '2014-07-20 22:15:28 +0200'
  date_gmt: '2014-07-20 20:15:28 +0200'
  content: Call me weird, but what does the User Entity have to do with the security
    component? Unless you store your user entity in the session as user object, they
    are not related... And I really hope you're not storing your Entity in your session
    ;)
- id: 1168
  author: Tobias Nyholm
  author_email: tobias@happyr.com
  author_url: ''
  date: '2014-07-21 06:52:37 +0200'
  date_gmt: '2014-07-21 04:52:37 +0200'
  content: "hehe. No, Im not storing my User *entity* in the session.  =) But of course
    you store some keys to be able to know which user is logged in.\r\n\r\nIf I wanted
    to do like @Brian suggested I need to make sure to add the extra JOIN somewhere.
    I have to implement my own UserProvider and make sure it extends Symfony\\Component\\Security\\Core\\User\\UserProviderInterface.
    Because it is the UserProvider that loads the User object at each request."
- id: 1173
  author: Andras Ratz
  author_email: ratz.andras86@gmail.com
  author_url: ''
  date: '2014-07-21 09:53:09 +0200'
  date_gmt: '2014-07-21 07:53:09 +0200'
  content: When you use the Security to fetch a user you get back generally only the
    User entity, and if you have OneToOne relationships, doctrine will do additional
    ones. I had this same problem and solved the same way, for me it was more disturbing,
    when I just wanted to list users, but I didn't needed the - in this case called
    Resumse - additional info, but after 10 user in the list, doctrine did 10 more
    additional query, and I simply didn't wanted to fetch those info with a join cause
    I didn't needed them.
---

Many times I've come got to a situation where I have a unusual high query count. When I inspect the queries in the Symfony profiler I can see that Doctrine is fetching objects I have not requested. To give you a clear image of the problem I'll show an example of the database mapping.


{% highlight php %}


class User {
  /**
  * @ORM\Column(name=&quot;id&quot;, type=&quot;integer&quot;)
  * @ORM\Id
  * @ORM\GeneratedValue(strategy=&quot;AUTO&quot;)
  */
  private $id;


  /**
  * @ORM\OneToOne(targetEntity=&quot;Resume&quot;, mappedBy=&quot;user&quot;)
  */
  private $resume;
}


class Resume {
  /**
  * @ORM\Column(name=&quot;id&quot;, type=&quot;integer&quot;)
  * @ORM\Id
  * @ORM\GeneratedValue(strategy=&quot;AUTO&quot;)
  */
  private $id;


  /**
  * @ORM\OneToOne(targetEntity=&quot;User&quot;, inversedBy=&quot;resume&quot;)
  */
  private $user;
}
{% endhighlight %}


This will create SQL like this:


{% highlight sql %}
CREATE TABLE Resume (id INT AUTO_INCREMENT NOT NULL, user_id INT DEFAULT NULL, UNIQUE INDEX UNIQ_6602EC1AA76ED395 (user_id), PRIMARY KEY(id))
CREATE TABLE User (id INT AUTO_INCREMENT NOT NULL, PRIMARY KEY(id))
ALTER TABLE Resume ADD CONSTRAINT FK_6602EC1AA76ED395 FOREIGN KEY (user_id) REFERENCES User (id)


{% endhighlight %}


The relation information is on the Resume table because the Resume entity is on the owing side.

<h2>Fetching the relation eagerly</h2>

As a consequence of Resume being on the owning side of the OneToOne relation it will always be fetched eagerly. You can not fetch it as <em>lazy</em> (default for other relations) or <em>extra lazy</em>. This means that every time you fetch the user entity Doctrine will make a second query to fetch the Resume object. This could be the desired behavior with some relations but not in this case. It is reasonable to think that most of the the you may use the User object without the Resume.

<h2>Owning Side and Inverse Side</h2>

There is a few things to know about the owning side and the inverse side.

<ul>
<li>The owning side has the <em>inversedBy</em> attribute.</li>
<li>The inverse side has the mappedBy attribute.</li>
<li>Doctrine will only check the owning side of an association for changes.</li>
<li>The owning side of a OneToOne association is the entity with the table containing the foreign key</li>
</ul>

Source: <a href="http://doctrine-orm.readthedocs.org/en/latest/reference/unitofwork-associations.html">Doctrine docs</a>

<h2>The fix</h2>

The fix for this problem is simple. You just change the owning side and inverse side. Make the User entity the owning entity. It makes sense to eagerly fetch the User when you fetch a Resume, right?


So, the rule of thumb here is:

<blockquote>
The entity that owns the relation does not need the relation. The inverse side is depended on the relation and will fetch it's owner.
</blockquote>

&nbsp;


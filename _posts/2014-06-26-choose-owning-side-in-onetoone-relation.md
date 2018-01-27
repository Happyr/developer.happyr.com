---
title: Choose owning side in OneToOne relation
author: Tobias Nyholm
date: '2014-06-26 12:16:41 +0200'
categories:
- Symfony2
- Performance
- Doctrine
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


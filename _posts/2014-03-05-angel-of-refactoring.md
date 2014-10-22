---
layout: post
title: "The Angel of Refactoring"
tags:
  - refactoring
status: publish
type: post
published: true
---
This is one of the nicest feelings in coding. You've been working a couple of
hours on some legacy code, with a thorny refactoring of stuff that's used in
a couple of places. After a morning buried in the code that deals with one of the implementations, you have a nicely
reworked version, and you're at a few hundred lines of delta.

<!-- more -->

You realize you now need to implement the new solution in the other place where this problem is addressed. That code, you remember, is a bunch of hacked-together smelly rubbish. You grumble, wondering how much more work this is going to take.

But the Angel of Refactoring is waiting for you in the other spot. You look at the code, and realize that the new version has this case covered without
contorsions, and this:

{% highlight php startinline %}
$presenter = $this->getPresenter();
$presenter->setProjects(array($project->toArray()));
$collection = json_decode(json_encode($presenter->handle()), true);
$this->result = $this->apiLinkTransformer->transform($collection[0]);
{% endhighlight %}

... becomes this:

{% highlight php startinline %}
$this->result = $this->getProjectDecorator()->decorateProject($project);
{% endhighlight %}

And all you can say is: XD

Thank you, Angel of Refactoring!


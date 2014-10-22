---
layout: post
title: "Why Laravel Homestead Makes Me Nervous"
tags:
  - Laravel
  - devops
  - architecture
status: publish
type: post
published: true
---
I had an immediate negative reaction to the news of Laravel's new Homestead program.

In [their own words](http://laravel.com/docs/homestead?version=4.2#introduction):

> Laravel Homestead is an official, pre-packaged Vagrant "box" that provides you a wonderful development environment without requiring you to install PHP, a web server, and any other server software on your local machine.

For starters, part of my reaction was due to my irritation at all the marketing that Laravel has been pushing out regarding an upcoming announcement, which I presume referred to Homestead, and Forge, their new hosting solution. My perception of that as hype, in addition to the not-terribly-innovative nature of this huge revelation, didn't dispose me well to the announcement.

In truth, I can see how the availability of this pre-built development platform will be a tremendous boon to a lot of Laravel users.

Now that I've had a chance to get past the initial emotional reaction, I can start to analyze the more objective reasons this project bothers me.

## When I was your age

I've been doing web development with PHP for about 6 years. In that time, I've bootstrapped my knowledge of Linux in general, and running servers in particular. I learned, from the ground up, all about configuring Ubuntu servers, including using Puppet for management. Since I joined the Imagine Easy team in January, I've added some experience with cloud virtual-server hosting and deployment.

This in-the-trenches learning is both a reflection of my natural mode of "dive in there and figure it all out," and a reinforcement of it. Most of my experience has come with the luxury of time and resources to just dig in and get to know different ways of solving problems in a given space. This means that I value getting my hands dirty with all different parts of the system, and having a hand in setting everything up.

The notion of a pre-built environment is a challenge to that perspective. Whereas I try to get as much perspective and knowledge as I can to inform decisions about my stack, I see Homestead as taking that away from developers. This means they are trained to delegate decision-making, as well as losing the actual knowledge and in-depth experience they might have gotten by being thrown in the deep end.

I *fully acknowledge* that this is not a complete nor totally fair assessment of Homestead; but this outlook greatly influenced the light in which I see Homestead.

## The Laragarden

<blockquote class="twitter-tweet" lang="en"><p><a href="https://twitter.com/yitznewton">@yitznewton</a> Laragarden</p>&mdash; Michael Hasselbring (@mikelbring) <a href="https://twitter.com/mikelbring/statuses/467059489405812736">May 15, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

A more serious argument against buying into Homestead is the notion of platform dependency. Over the last two years, Robert C. Martin ("Uncle Bob") has been by far the greatest influence on my thinking. One of his mantras (see [Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html)) is maintaining independence from things that you don't control. In the context of frameworks, it means isolating your business logic from the framework in discrete, Plain Old Object code. One immediate benefit of this is that the core logic is easier to test, being separated from any framework plumbing. A second is that, when the framework inevitably changes, you are not forced to make painful changes in the midst of your application; instead, all that's necessary is to write the boundary layer that connects your free-floating business code to the framework.

The same concept applies to other components of the stack. If you hard-code persistence into your core code in terms of (MySQL\|MongoDB\|ActiveRecord\|Redis) or whatever, and mix business logic in the same place as persistence, you are stuck with that decision, unless you want to rewrite it.

Using a full-stack boxed solution like Homestead encourages that sort of thinking. I acknowledge 100% that it doesn't *force* the behavior of lazily (or unwittingly) coupling these external elements into your application, but it almost implies that is the way to go.

## Down the road

Furthermore, let's say an application grows, and requires a particular piece added to the stack -- Solr search, perhaps. How easy will it be to add those components? Will the knowledge gap fostered by Homestead bite the project developers now that they need to extend the stack? You run into the same fragility problem where decisions made by the Homestead team down the road may conflict with your local emendments to the build.

The project's documentation also [indicates](http://laravel.com/docs/homestead?version=4.2#daily-usage) that running multiple Laravel applications on a single Homestead instance is the normal mode of operation; that seems to indicate that the Homesteaders are not taking project-specific build modifications into account.

## Conclusion

I am confident that Laravel Homestead will help a lot of people, and I am grateful to the entire Laravel enterprise as a key player in the continuing maturation of the PHP community. I have tried to highlight what I see as the key shortcoming of Homestead, and why I believe sober discretion is required in using such a pre-boxed solution.

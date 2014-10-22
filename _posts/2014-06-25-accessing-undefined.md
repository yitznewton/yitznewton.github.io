---
layout: post
title: "Accessing undefined properties of hashes/objects (in PHP and more)"
tags:
  - PHP
  - syntax
  - language comparison
  - data structures
status: publish
type: post
published: true
---
The true focus of this post is how to retrieve given member values of associative arrays in PHP, but it will touch on analagous constructs in other popular languages, namely Java's `Map`, Python's `dict`, Ruby's `Hash`, and Javascript objects.

<!-- more -->

The operation can be characterized as:

> Given an object **Foo**, if the **Bar** property is set, return that; otherwise return a default value **Default**

This is such a common use case that I would expect this to have decent syntax support in any language that deals with open-ended maps such as the above.

**Disclaimer**: I'm not well-versed in all of these languages, so forgive any syntax blunders or missed opportunities.

**BTW**: One of the few "features" of PHP that I genuinely hate is the blurring of the map/list distinction that is the `array`. I'm psyched that Hack is trying to [fix this](http://docs.hhvm.com/manual/en/hack.collections.php).

## Java

[`Map.get(key)`](http://docs.oracle.com/javase/7/docs/api/java/util/Map.html#get(java.lang.Object)) returns the value for the requested key, or `null` if it is not set. This has the unfortunate consequence of a logic flaw if `null` is a possible member of the `Map` (which is a separate can of worms). So the implementation would still have to be:

{% highlight java %}
foo.containsKey(bar) ? foo.get(bar) : default;
{% endhighlight %}


## Python

Python's [`dict`](https://wiki.python.org/moin/KeyError) hooks us up proper. Full stop.

{% highlight python %}
foo.get('bar', default)
{% endhighlight %}

## Ruby

[`Hash`](http://www.ruby-doc.org/core-2.1.2/Hash.html#method-i-default) is a curious beast on this question. A default value can be set *on the Hash itself* as an instance setting, rather than as part of a specific retrieval call. This works either as an argument passed to the constructor, or via the setter `#default=`:

{% highlight ruby %}
foo = Hash.new(default)
# or
foo.default = default

foo[:bar]  # returns default
{% endhighlight %}

### Original Ruby conclusion

To me, employing that technique in this use case seems like a misleading use of that instance setting, so I would be inclined to do:

{% highlight ruby %}
foo.has_key?(:bar) ? foo[:bar] : default
{% endhighlight %}

### Updated Ruby conclusion

Tim Morgan in the comments helpfully pointed out the method I was looking for, namely `Hash#fetch`. So Ruby and Python are neck-and-neck:

{% highlight ruby %}
foo.fetch(:bar, default)
{% endhighlight %}

## JavaScript

JavaScript and its `undefined` property is its own mess, so I'll just say that JS [gives you the `in` operator](http://stackoverflow.com/a/1098955/614709):

{% highlight javascript %}
("bar" in foo) ? foo.bar : default
{% endhighlight %}

## PHP

Which brings us to the topic of this post. Similarly to Java, when accessing nonexistent members of an associative array in PHP, [`null` is returned, but with an `E_NOTICE` error](http://us1.php.net/manual/en/language.types.array.php#language.types.array.syntax.accessing). So, leveraging the beautiful [Elvis operator](http://en.wikipedia.org/wiki/Elvis_operator) (one of my favorite syntactic sugar cubes), we get:

{% highlight php startinline %}
$foo['bar'] ?: $default  // throws E_NOTICE
{% endhighlight %}

To bypass the logic flaw of the `null`, we "can often assume" that `null` is not a possible value for the array in question. I had (or saw someone employ) the brilliant idea of using PHP's error suppression operator, `@`, to make this behave like the Java version, i.e. not throw the `E_NOTICE`:

{% highlight php startinline %}
@$foo['bar'] ?: $default  // E_NOTICE is suppressed
{% endhighlight %}

Obviously suppressing errors is a controversial matter. Some would argue that it's altogether poor style. It's also a slippery slope, especially in a team setting: this is the only scenario where I would consider using `@`, but it's hard to enforce that discipline across a team. So, I posted this solution on our team chat for feedback.

We are fortunate enough to have [Nils Adermann](https://twitter.com/naderman) on our team, and he weighed in with some more fundamental concerns, aside from the question of code style.

### The bad

> Imagine the `@$params` being an array access object which triggers autoloading in one of its functions; then `@$params['foo']` would hide parse errors in those PHP files. So it's a generally bad idea, since you can never be quite certain what else you are accidently silencing in addition to what you mean to silence.

> Furthermore, `@` is actually slow, since it triggers the error, jumps into the error handler, executes code there to figure out it's supposed to be silent, and only then returns back to the original code.

Ha! So, to address the first concern, if we are disciplined we can only use this practice on the sorts of plain `string => string` arrays that had conceived of; but that doesn't solve the performance concern. And it requires discipline.

### The alternative

We're also fortunate to have [Richard Wossal](https://twitter.com/r_wos) on board. He pointed out a function library that provides this functionality: [igorw/get-in](https://github.com/igorw/get-in).

{% highlight php startinline %}
\igorw\get_in($foo, ['bar'], $default);
{% endhighlight %}

This is clunkier than my ill-conceived PHP version, or the Python version, but it does what we need. Additionally, as of [PHP 5.6](http://www.php.net//manual/en/migration56.new-features.php#migration56.new-features.use), we can `import` functions, so this would become that much more fluid (without the inline fully-qualified name).

The use of this function together with anonymous functions also lends itself to creative local solutions:

{% highlight php startinline %}
function extractParts(array $data)
{
    $part = function ($key) use ($data) {
        return \igorw\get_in($data, [$key]);
    };

    return [
        $part('family'),
        $part('given'),
        $part('suffix'),
    ];
}
{% endhighlight %}

## Conclusion

This was a really nice way to explore the powers and limitations of this data structure in various languages, and seeing the implications of pushing those limits. It was also a good illustration of how ideas that smell questionable, very likely are questionable.

## Addendum

Here's my interpretation of Go, using the "comma ok" double-return-value idiom:

{% highlight go %}
if bar, ok := foo["bar"]; ok {
    return bar
} else {
    return default
}
{% endhighlight %}

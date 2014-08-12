---
layout: post
title: "Pushing the limits of metaprogramming in PHP: aspect oriented design"
tags:
  - PHP
  - AOP
status: publish
type: post
published: true
---
# Pushing the limits of metaprogramming in PHP: aspect oriented design

I've been doing software for about six years professionally. It was a few years into my career that I first heard about Aspect Oriented design (AOP == Aspect Oriented Programming is your buzzword).

Here's the premise. A given piece of code exists for a certain purpose - let's say, to retrieve a record from a database. But there may be any number of other things that need to happen in addition to the actual retrieval: logging, access control, caching... those are known as *cross-cutting concerns* -- issues that are relevant across the codebase, but are not specifically relevant to any one piece of code where they might be needed. And being that these bits of functionality are not intrinsically connected with data retrieval, in our example, it would make sense for them to be disconnected from the retrieval implementation.

_**Pedantic disclaimer**: the following code is meant for illustrative purposes only; any design issues are beside the point_

So if I wanted to log the retrieval, I could do:

{% highlight php startinline %}
class UserController
{
    public function show($userId)
    {
        Logger::debug('retrieving user record ' . $userId);
        $user = $this->repository(User::class)->fetch($userId);
        Logger::debug('retrieved user record ' . $userId);
        
        return $this->renderView($user);
    }
}
{% endhighlight %}

But what if you could remove that clutter?

{% highlight php startinline %}
public function show($userId)
{
    $user = $this->repository(User::class)->fetch($userId);
    return $this->renderView($user);
}
{% endhighlight %}

...and put the logging somewhere else:

{% highlight php startinline %}
/**
 * @Aspect.before(Logger, debug, 'retrieving user record {$userId}')
 * @Aspect.after(Logger, debug, 'retrieved user record {$userId}')
 */
public function show($userId)
{
    $user = $this->repository(User::class)->fetch($userId);
    return $this->renderView($user);
}
{% endhighlight %}

...or even better,

{% highlight php startinline %}
class UserController
{
    public function show($userId)
    {
        $user = $this->repository(User::class)->fetch($userId);
        return $this->renderView($user);
    }
}

aspect Logging
{
    UserController::show::before($userId)
    {
        Logger::debug('retrieving user record ' . $userId); 
    }

    UserController::show::after($userId)
    {
        Logger::debug('retrieved user record ' . $userId); 
    }

    UserController::show::exception($userId, Exception $e)
    {
        $message = sprintf(
            'failed to retrieve user record %s: %s',
            $userId,
            $e->getMessage()
        );

        Logger::error($message);
    }
}
{% endhighlight %}

Obviously that cool little `aspect Logging` thing is total pseudocode, but it's easy to see the power of pulling the logging concern out.

## Caching methods

My real-world use case and motivation to explore the possibilities of AOP relates to caching an expensive method call. Whether it's a call to a remote server, or a complex calculation, sometimes there is an operation whose result you want to store within the code that requests it, rather than have to run the operation again if the result is needed a second time. A typical implementation might be:

{% highlight php startinline %}
class ComplexCalculation
{
    private $calculatedValue;

    public function calculate()
    {
        if ($this->calculatedValue) {
            return $this->calculatedValue;
        }

        // do all sorts of complex stuff

        return $this->calculatedValue = $calculatedValue;
    }
}
{% endhighlight %}

But storing the result as an instance variable introduces the need to create, check and set that variable -- another 4.5 lines of code, which hinders the ability to quickly understand the code, and adds to the cost of maintaining it. And you need to repeat it for every method you want to cache, across your codebase.

Why can't we just do this:

{% highlight php startinline %}
class ComplexCalculation
{
    /**
     * @CacheResult
     */
    public function calculate()
    {
        // do all sorts of complex stuff

        return $calculatedValue;
    }
}
{% endhighlight %}

Now we have a single, central place for the caching logic to live: the implementation of the `CacheResult` aspect. The main implementation is also cleared of extraneous concerns. So how to implement this?

## One approach in PHP

The [Lithium framework](https://github.com/UnionOfRAD/lithium) makes extensive use of closures to add functionality dynamically. [Lithium's manual](https://github.com/UnionOfRAD/manual/blob/master/jp/02_lithium_basics/02_filters.md#aspect-oriented-programming) describes the methodology: a filter chain is used to expose specific framework calls (as well as calls within apps built on Lithium) for extension.

Here is Lithium's implementation of a logging aspect applied to the `Connections::_execute()` method:

{% highlight php startinline %}
use lithium\analysis\Logger;
use lithium\data\Connections;

// Filter the database adapter returned from the Connections object.
Connections::get('default')->applyFilter('_execute', function($self, $params, $chain) {
    // Hand the SQL in the params headed to _execute() to the logger:
    Logger::debug(date("D M j G:i:s") . " " . $params['sql']);
    
    // Always make sure to keep the filter chain going.
    return $chain->next($self, $params, $chain);
});
{% endhighlight %}

Now that looks kind of ugly, but theoretically, instead of using an anonymous function, we could insert a `[CachingAspect::class, 'filter']` style callable, which would be reusable.

{% highlight php startinline %}
// the aspect class:

class CachingAspect
{
    public static function filter($self, $params, $chain)
    {
        if (self::hasCachedValue($self, $params)) {
            // inject $cached into the $params, which get carried down the
            // filter chain
        }
        else {
        }
        
        return $chain->next($self, $params, $chain);
    }
}

// register the aspect:

UserRepository::applyFilter('fetch', [CachingAspect::class, 'filter']);
{% endhighlight %}

Or something like that... my Lithium-fu is rusty. And therein lies the fail. I worked with Lithium for a little over a year; my experience led me to avoid the filter system. The most undesirable aspect (heh, no pun intended) was difficulty in understanding program flow: because closures can be tossed in anywhere, responsibility for function can vary unpredictably - an enemy of comprehension and testability. Because so much functionality can be contained in anonymous closures, backtraces have very little specific information to aid in isolating problems, and debugging tools (Xdebug) are not always able to handle the closures. Similarly, when I attempted to profile an app of modest-to-medium size with [XHProf](http://pecl.php.net/package/xhprof), the call stack was full of thousands of calls to nested filters, which included a network of, you guessed it, anonymous functions.

In writing the filters, you have to maintain the filter chain, and get it right whether your filter goes before or after the `$chain->next()` call. As you can see from the example, there's also a lot of baggage that comes along with the use of closures: three context variables are passed into each closure, and require manipulation and passing along. More cognitive overhead. The slippery slope of closure spaghetti made me loath to actually use the technique; it was a razor thin line between power and abuse.

![Don't tempt me!](https://31.media.tumblr.com/20277559083f319b000f11e8ab9d1ac1/tumblr_inline_n190kmqcsP1qhp1cd.gif)

> With that power I should have power too great and terrible.
>
> And over me the code would gain a power still greater and more deadly... Do not tempt me!

## Arriving at a spec

So rather than globbing on functionality dynamically via closures, I'm thinking subtypes. Here's what I'm looking for in a prospective aspect-oriented cache. Given a method call that will always return a consistent value for any given combination of arguments:

* The caching solution must always return a consistent value for each combination of arguments.
* The solution must call the cached method exactly one time (the first time), and thereafter return the original return value from cache.
* I must be able to [substitute](http://en.wikipedia.org/wiki/Liskov_substitution_principle) the caching version and the original non-caching version interchangeably. As the consumer of the caching version, I should not be able to tell the difference, and type hints should continue to work.
* I should be able to choose between different caching backends.

## Looking beyond PHP

As it happens, there is a popular third-party package for Ruby called [cache_method](https://github.com/seamusabshere/cache_method) which does exactly this. Let's see how they do it. The `cache_method` method itself is added to the class to be cached via a Ruby module, which functions here as a mixin (similar to a [PHP Trait](http://www.php.net/manual/en/language.oop5.traits.php)).

{% highlight php startinline %}
class Blog
    # cache that slow method!
    cache_method :entries

    def entries(date)
        # ...
    end
end

#####################################################################

def cache_method(method_id, ttl = nil)
    original_method_id = "_cache_method_#{method_id}"
    alias_method original_method_id, method_id
    define_method method_id do |*args, &blk|
        ::CacheMethod::CachedResult.new(self, method_id, original_method_id, ttl, args, &blk).fetch
    end
end
{% endhighlight %}

So `cache_method` "copies" the main implementation to a new method, and creates a new method which (presumably) composes the original one like a [Proxy](http://en.wikipedia.org/wiki/Proxy_pattern), calls it the first time, and hits the cache for subsequent calls.

Wait... what? Creates a new method? So here's where Ruby's object model comes in handy. In Ruby, classes are full-fledged objects, so you can manipulate them at runtime. (I'm sure Ruby experts will qualify that statement, but let's assume it's essentially true.) In PHP, on the other hand, once a class is created, it cannot be changed. The best you can do is use PHP's [Reflection](http://www.php.net/manual/en/book.reflection.php) capabilities to *examine* the class.

## Bringing the lesson back home

Since we can't actually substitute a proxy method for the original `ComplexCalculation::calculate()`, we are stuck with inheritance if we want to be able to substitute transparently.

There are two ways of arriving at this point programatically, and both involve generated code. Either we add a generation step to the build process, outputting actual files that will live somewhere under our project's file structure, or we generate the classes on the fly, with the help of the dread function `eval()`. Following the path of emulating the Ruby library, and given my distaste for cluttering up the project with generated code, I opted to investigate the possibility of on-the-fly generation.

For the first attempt, I thought about another situation where we substitute functionality within an existing class: test stubs -- specifically, the PHPUnit mock framework. I discovered that PHPUnit uses `eval()`ed generated code. I thought about leveraging those mocks for this project, but I realized that, while it's possible to set a return value for a given mock extension of a method, this application would require varying the return value according to the method parameters. That seemed to be an overextension of PHPUnit mocks, which would require smelly workarounds to wrap the normal `returnValue` functionality.

## The solution

At this point, [Camphor](https://github.com/yitznewton/camphor) was born. In this project, I was able to use PHP reflection to gather a couple of pieces of data about the method(s) being cached, in order to `string` together a class that extends the class being cached.

The heart of it is [here](https://github.com/yitznewton/camphor/blob/master/src/EasyBib/Camphor/CacheAspect.php#L45). Ultimately, it's just an implementation of the Proxy pattern, where the class is assembled on the fly; the added value of the library is that the proxy does not need to be re-created for each class under cache. Here's a sample of usage:

{% highlight php startinline %}
use EasyBib\Camphor\CacheAspect;
use EasyBib\Camphor\CachingProxy;

$cachingProxy = new CachingProxy();
$cacheAspect = new CacheAspect($cachingProxy);
// register the Foo class and cache the bar method
$cacheAspect->register(Foo::class, ['bar']);

$myFoo = new CachingFoo();

$firstCall  = $myFoo->bar('jimmy');  // calls Foo::bar('jimmy')
$secondCall = $myFoo->bar('jimmy');  // retrieves the value from cache
$otherCall  = $myFoo->bar('billy');  // calls Foo::bar('billy');
{% endhighlight %}

I endeavored to keep as much code as possible in regular static PHP classes, because... well, there are weaknesses here. `eval()` is generally considered evil, and although there is no use of end-user-provided values in this eval, the classes still fall outside of any opcode cache being used. My colleague [captured my sentiments pretty well](https://github.com/yitznewton/camphor/pull/1#issuecomment-35140877): it's an interesting idea and I am very glad I conducted this experiment, but I am uncomfortable about the idea of using this apparatus in production code. Ultimately I decided to move forward with a traditional Proxy pattern for the actual solution to my caching problem.

Have any more reasons why using this in production would be particularly good or bad?

![Surprisingly OK](https://31.media.tumblr.com/19017a6efbc6a8b8d12be01cc28ae7f9/tumblr_inline_n190jndjdc1qhp1cd.jpg)

Let us know in the comments!

## Epilogue

So, after going through this project, I discovered that since my last research expedition a couple years back, there's a new, apparently legit AOP framework for PHP, called [**Go!**](https://github.com/lisachenko/go-aop-php) Check this out, from [a tuts+](http://code.tutsplus.com/tutorials/aspect-oriented-programming-in-php-with-go--net-31046):

{% highlight php startinline %}
class BrokerAspect implements Aspect {
    /**
     * @param MethodInvocation $invocation Invocation
     * @Around("execution(public Broker->*(*))")
     */
    public function aroundMethodExecution(MethodInvocation $invocation) {
        $returned = $invocation->proceed();
        echo "method returned: " . $returned . "\n";
 
        return $returned;
    }
}
{% endhighlight %}

The API appears to be pretty much exactly as I envisioned. In terms of implementation, as far as I can tell, it uses a hybrid file-generation/on-the-fly model: it generates Proxy classes, outputs them as generated files at runtime, and `include`s them. I didn't finish delving its depths, but I have a feeling it substitutes generated, aspect-ified classes based on the originals, via some tricky autoloading footwork. The concept is extremely similar to my model, but fully-baked. I prefer the lightness of Camphor in both usage and implementation (no heavy bootstrapping), but I think Go! clearly has the advantage in production-worthiness. It also does not introduce the differently-named class, `CachingFoo`.

I'm still happy I did this experiment, but the moral of the story is: make sure your ecosystem knowledge is up-to-date before assuming that your solution doesn't already exist!

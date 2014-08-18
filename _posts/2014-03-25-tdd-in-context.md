---
layout: post
title: "Test-Driven Development in Context"
tags:
  - TDD
status: publish
type: post
published: true
---
This article comes in response to [a critique of TDD](http://dev.imagineeasy.com/post/79356891740/test-driven-development-is-not-the-solution) eloquently advanced by my colleague, [Richard Wossel](http://r-wos.org/). Rather than offer a point-by-point comparison of our perspectives, I am going to paint TDD in a broader context, and I hope that this will clarify its value as I see it. <!-- more -->

The most essential aspect of my practice of TDD to me, is that it *works* when I create software in this way. I look at my output, and I see that the quality improves when I test-drive. I have a better understanding of the process and the product. I also prevent myself from introducing bugs, and catch edge cases up front that would have needed post-hoc analysis to identify and correct. My code is better insofar as TDD makes me think more carefully about what I am doing, in order to write correct and comprehensive tests. This alone justifies the practice to me.

So what is TDD? I find it useful to think of it in terms of *Behavior-Driven Development*. Like [Mike Brown](http://programmers.stackexchange.com/users/13181/mike-brown) and subsequent commenters on [this Programmers Stack Exchange answer](http://programmers.stackexchange.com/a/135246/42950), I see BDD as nothing more than a clearer
restatement of Test-Driven Development. TDD represents a focus on defining and measuring behavior of the *system under test (SUT)*.

## Example application

We have a product to build; let's say that it is the web service API behind a mobile check-in app. We have decided on a framework, and are ready to begin development. How do we proceed? What guides us through the process of building the application? We should start by considering the response we get back from the API for a checkin. We expect that the response code will be 201, the body will consist of a particular JSON status, and the `Content-Type` header will correspond to this.

{% highlight gherkin %}
Given I have a logged-in user
When I receive a POST to /checkins/ with {lat: 23.45, lng: 98.76}
I should respond with a 201 status
And I should include the Content-Type header of application/json
And I should include a response body of {"status":"ok"}
{% endhighlight %}

Sound and look like acceptance tests? Exactly! TDD and "acceptance" tests intersect at this highest level. We can write those tests now, and if we need to, defer writing the implementation, marking them incomplete as we drill down into the application. For the sake of this post, I am going to assume that we have taken care of the "logged-in user" requirement. We can make this test pass easily enough (use your imagination with regard to the framework):

{% highlight php startinline %}
public function createAction()
{
    return (new Response())
        ->setStatus(201)
        ->setContentType('application/json')
        ->setBody(json_encode(['status' => 'ok'])
        ;
}
{% endhighlight %}

Let's consider the case of invalid coordinates. Assume we have built the above case already, with a `CheckinController` which has a `create` action. Here's an abbreviated test:

{% highlight gherkin %}
When I receive a POST to /checkins/ with {lat: foo, lng: 123.4}
I should respond with a 400 status
{% endhighlight %}

When we start writing the `CheckinController:create` action, we discover that some code somewhere needs to know about valid and invalid coordinates. For the sake of passing the tests, we could place that in the controller, and extract to its own class later. With the following, we should be able to make the above tests pass:

{% highlight php startinline %}
public function createAction()
{
    $coordinates = $this->request->getParams();
  
    if (!$this->validateCoordinates($coordinates)) {
        return (new Response())
            ->setStatus(400)
            ;
    }
    
    return (new Response())
        ->setStatus(201)
        ->setContentType('application/json')
        ->setBody(json_encode(['status' => 'ok'])
        ;
}

private function validateCoordinates(array $coordinates)
{
    if (!is_float($coordinates['lat'])) {
        return false;
    }
    
    return true;
}
{% endhighlight %}

This would be a good time to extract that validation into its own class -- the "refactor" step of the "red-green-refactor" cycle. Now arriving at the creation of a coordinate validator, we have the ideal opportunity to do our initial thinking about edge cases. We can leave our 400 error test above to test controller behavior, and bring the case itself down as the first unit test for the validator:

{% highlight gherkin %}
When I receive a latitude which is a string and a longitude which is a float
Then I should return false
{% endhighlight %}

Since we need concrete test cases, we will need to choose specific values, being careful that there are no assumptions which gloss over further "typey"  edge cases, such as floating-point math slop, truthy-falsy inconsistencies, and the like. Here we translate this test into xUnit syntax:

{% highlight php startinline %}
$this->assertFalse($validator->validate(['lat' => 'foo', 'lng' => 123.4]));
{% endhighlight %}

## Analysis

Based on these examples, we can derive some conclusions about TDD. Firstly, it is not inherently about the "unit." The focus can be anything from a minor subsystem responsible for validating a particular type of input, to the entire application. It's also not specifically about finding or preventing bugs. The goal is ensuring that you have defined exactly what your unit of code is supposed to *do* before writing it, and that the code in fact *does those things, and nothing else.* It's a general set of practices that informs the development process for all levels of an application.

TDD carries several immediate design benefits. Firstly, it uses human nature to push the developer to manage dependencies and coupling. The insistence on testability means that we need to set up each system for testing. The more dependencies a SUT has, the more irritating the process of building and maintaining the tests. The ongoing frustration that comes with having to manage excessive dependencies helps spur the developer to appropriately divide responsibilities. This leads to the system being flexible enough to accomodate future growth and changes in the application, and *this* is a direct design benefit of TDD.

Secondly, because all changes must be preceded by tests, it follows that, if a behavior cannot be tested, it cannot be incorporated into the system. This ensures that the magical and mystical, along with their attendant mystical bugs, will be absent.

An additional design benefit results from the outside-in behavioral orientation that I have assumed. Because we always look at a system under test from the perspective of the consumer of that system, we are more attuned to considerations of interface, which helps us choose better names. We also get direct and immediate feedback on the way we have structured the divisions between subsystems.

## Conclusion

I am not a strict TDD practitioner (yet), but I have been working these techniques into my coding over the past several years, and I am still learning and refining them. I do it because I perceive the benefit in the code I create. I would consider my main mentor in this to be [Uncle Bob Martin](http://blog.8thlight.com/uncle-bob/archive.html), via his writings. I encourage anyone with further interest to read the excellent material on TDD in his classic [Agile Software Development, a.k.a. "the PPP book"](http://www.amazon.com/Software-Development-Principles-Patterns-Practices/dp/0135974445).

## Epilogue

I used a pseudo-framework for the examples above, but the outside-in design methodology could apply here also: we could continue on building the framework itself based on the interface decisions brought on by our demo application.

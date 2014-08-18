---
layout: post
title: "Learn You a Haskell for Great Good in PHP, Ruby, ..."
tags:
  - functional programming
  - recursion
  - haskell
  - php
status: publish
type: post
published: true
---
My main production language is PHP. I started dabbling in Haskell a couple of years ago, out of curiosity. I've found a few
[great](http://learnyouahaskell.com/)
[resources](http://www.seas.upenn.edu/~cis194/lectures.html)
[out there](http://book.realworldhaskell.org/)
and have gone through the novice lessons a few times. I still haven't graduated to the point of building Real Things in the language; I *have* noticed, however, that exposure to the very different approach and style of Haskell had an immediate broadening effect on my PHP work. <!-- more --> I encountered a great example of that yesterday.

## Iterating with exceptional cases

The problem: we need to collapse a list of strings by interposing a delimiter, the classic sort of `implode()` operation. But not all cases get the delimiter. The actual application we are building allows for different rules, but let's treat the "no-Oxford-comma" case for the purposes of this blog post.

{% highlight php startinline %}
class RenderDelimitedTest extends \PHPUnit_Framework_TestCase
{
    public function data()
    {
        return [
            [
                [
                ],
                '',
            ],
            [
                [
                    'jim',
                ],
                'jim',
            ],
            [
                [
                    'jim',
                    'bob',
                ],
                'jim and bob',
            ],
            [
                [
                    'jim',
                    'bob',
                    'joe',
                ],
                'jim, bob and joe',
            ],
            [
                [
                    'jim',
                    'bob',
                    'joe',
                    'pete',
                ],
                'jim, bob, joe and pete',
            ],
        ];
    }

    /**
    * @dataProvider data
    * @param string[] $strings 
    * @param string $expected 
    */
    public function testRenderDelimited(array $strings, $expected)
    {
        $this->assertEquals($expected, render_delimited($strings));
    }
}
{% endhighlight %}

If we were to use a loop for this, each iteration would have to know the context of the looping in order to know whether the delimiter should be added. To me, that is both a potential breeding-ground for bugs, and ugly; I should be able to *declare* the correct rendering for a given segment or combination of segments without reference to segments that I'm not working on.

## Pattern matching and recursion

In Haskell, **pattern matching** is one of the main ways to choose between alternate implementations, given your input. This problem could be solved in Haskell as follows. The function is defined with a type signature first, and then the actual function definition.

{% highlight haskell %}
renderDelimited :: [String] -> String
renderDelimited [] = ""
renderDelimited [a] = a
renderDelimited [a,b] = a ++ " and " ++ b
renderDelimited (a:theRest) = a ++ ", " ++ renderDelimited theRest
{% endhighlight %}

(I've bastardized Haskell conventions a little for the sake of unfamiliar readers; Haskellians, forgive me!)

The `renderDelimited` function takes an array of `String`s, and returns a `String`. There are four cases: the empty array, a one-member array, a two-member array, and "everything else," in the form of "`a` merged with `theRest`."

I've applied **recursion** to build the full string. Where there are three or more members, we render the first member, the delimiter, and then *the output of rendering the rest of the list with the same function.* That's what makes it recursive.

{% highlight haskell %}
*Main> renderDelimited []
""
*Main> renderDelimited ["bob"]
"bob"
*Main> renderDelimited ["bob", "pete"]
"bob and pete"
*Main> renderDelimited ["bob", "pete", "joe"]
"bob, pete and joe"
*Main> renderDelimited ["bob", "pete", "joe", "larry"]
"bob, pete, joe and larry"
{% endhighlight %}

PHP obviously doesn't support pattern matching in this way, but we can do the same thing with explicit conditionals. Use of the `implode()` function allows us to merge the `1` and `2` cases.

{% highlight php startinline %}
/**
 * @param array $strings 
 * @return string
 */
function render_delimited(array $strings)
{
    $count = count($strings);

    if ($count === 0) {
        return '';
    }

    if ($count < 3) {
        return implode(' and ', $strings);
    }

    return $strings[0] . ', ' . render_delimited(array_slice($strings, 1));
}
{% endhighlight %}

Boom! The tests pass.

By using this technique drawn from bread-and-butter Haskell, we've avoided bringing the surrounding context explicitly into the iterations of a loop, and instead we are evaluating each step on its own.

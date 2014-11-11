---
layout: post
title: "Testing Logging in Silex"
tags:
  - php
  - silex
status: publish
type: post
published: true
---
[Silex](http://silex.sensiolabs.org/) is a PHP microframework from the same family as [Symfony](http://symfony.com/). My shop, [Imagine Easy Solutions](http://www.imagineeasy.com/), uses Silex for some of our most important applications.

Modular setup is at the core of Silex's game, by means of Service Providers. The [MonologServiceProvider](http://silex.sensiolabs.org/doc/providers/monolog.html) makes it easy to add highly configurable logging to your application.

But how to test your logging? It turns out that this Service Provider includes a `DebugHandler` which you can use to make log entries available in array form.

<!-- more -->

The complete sample app for this post is available at https://github.com/yitznewton/silex-log-test

For the purposes of this post, we'll assume you've already set up logging -- you can compare your `composer.json` and bootstrap with the ones in the sample repo. The question is, how to access the logs from your tests.

In the test setup, we need to inject a `DebugHandler` into our Silex container.

```php
namespace Yitznewton\SilexLogTest\Tests;

use Monolog\Logger;
use Silex\WebTestCase;
use Symfony\Bridge\Monolog\Handler\DebugHandler;
use Symfony\Component\HttpKernel\HttpKernel;

class WebTest extends WebTestCase
{
    /** @var DebugHandler */
    private $logHandler;

    public function setUp()
    {
        parent::setUp();
        $this->logHandler = new DebugHandler();
        $this->app['logger'] = new Logger('test', [$this->logHandler]);
    }
}
```

As an alternative, we can use the `DebugHandler` that comes pre-wired in the `MonologServiceProvider`:

```php
    public function setUp()
    {
        parent::setUp();
        $this->logHandler = $this->app['monolog.handler.debug'];
        $this->app['monolog']->pushHandler($this->logHandler);
    }
```

Now, we need to examine the collected logs so we have something to assert against:

```php
    private function assertLogEntry($level, $message)
    {
        $logMatches = function ($record) use ($level, $message) {
            if ($record['level'] != $level) {
                return false;
            }

            return strpos($record['message'], $message) !== false;
        };

        $records = array_filter($this->logHandler->getRecords(), $logMatches);
        $this->assertNotEmpty($records, 'Failed asserting that a log entry was made.');
    }
```

Now can test the presence of a log entry with a given log level, matching a given message.

```php
    /**
     * @test
     */
    public function notOk()
    {
        $client = $this->createClient();
        $client->request('GET', '/do-something/not-ok');
        $this->assertLogEntry(Logger::ERROR, 'woe is me!');
    }
```

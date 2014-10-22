---
layout: post
title: "Decoupling Domain Modules in AngularJS"
tags:
  - clean code
  - architecture
status: publish
type: post
published: true
---
The story and examples in this post are taken from my experience with Angular,
but from my cursory survey of popular JavaScript frameworks, the ideas
apply just as much to Ember.js and others.

I'm an avowed Uncle Bob-ist when it comes to... well, most things. In this
context, I have embraced his
[Clean Architecture](http://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html),
which describes why and how to
define and enforce boundaries between large-scale areas of responsibility within
an application, such as

* business logic
* frameworks (integration with HTTP, etc.)
* object storage (database abstraction layers, etc.)

<!-- more -->

<iframe src="//player.vimeo.com/video/43612849" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe> <p><a href="http://vimeo.com/43612849">Robert C. Martin - Clean Architecture</a> from <a href="http://vimeo.com/ndcoslo">NDC Conferences</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

## Wrecked Angular

This predilection led to an acute point of discomfort for me when my team
began working on a new project in AngularJS. This Model-View-Whatever
JavaScript framework offers its own dependency injection facility, and in code
examples, this DI service itself is injected into modules within the
application, so that they can in turn access their own dependencies.

This is the controversial **Service Locator** pattern, and here has an unfortunate
result that erodes the Clean Architecture.

**In allowing a business domain module to request its own dependencies via
Angular's DI service, you are giving knowledge about the framework to the
domain, and forever coupling them.**

The domain code should be the most core part of the application; in theory, it
could exist as part of a Node.js application on a server, or a hypothetical
Android.js application where Angular would not be used. By introducing Angular
into your code, you are preventing yourself from ever using it outside of an
Angular application. Such a coupling would also make it impossible to reuse
your domain code in e.g. Ember.js, in case Angular should die, or Ember introduce
some killer feature that you want to leverage.

Another outcome of having the framework baked into your domain is that you need
to load the framework in order to run your tests, which can be complicated and
can also erode performance. In a domain with relatively simple POJOs and few
dependencies, you should be able to run your tests fast and without fanfare.

So how do we break this coupling and allow our domain code to stand on its own
two feet?

The simple answer is to make sure that all the module's dependencies are
injected explicitly, typically as constructor parameters; however, I was unable
to find any examples of this mode of decoupled design on the webz.

## How to do it

With some JavaScript help from my colleague Cameron, I was able to pull together
a solution over the course of a couple days of intermittent experimentation.
Notice how the domain code (the "trelloboard/repository" module) has no knowledge of
Angular.

Our example application is a [Trello](http://trello.com) clone. I've
implemented a bare-bones repository object which returns `Trelloboard`
entities. Presumably it would look them up by ID, but in this naive
implementation, we're just hardcoding it.

Each `Trelloboard` has a number of `CardGroups`. We're using RequireJS for
loading.

The test injects the `q` dependency directly into the module, and the Angular
`services` module wires our module into the framework before the controller
needs it.

### The module

```js
// trelloboard/repository.js
define(function() {
  'use strict';

  function TrelloboardRepository($q) {
    this.$q = $q;
  }

  TrelloboardRepository.prototype = {
    find: function() {
      // first iteration: just return a hard-coded Trelloboard
      var output = this.$q.defer();

      output.resolve({
        cardGroups: [
          {
            name: 'Ungrouped',
            cards: []
          }
        ]
      });

      return output.promise;
    }
  };

  return TrelloboardRepository;
});
```

### The spec

```js
define(['trelloboard/repository', 'q'], function(TrelloboardRepository, q) {
  'use strict';

  var trelloboardRepository;

  describe('TrelloboardRepository', function() {
    beforeEach(function() {
      trelloboardRepository = new TrelloboardRepository(q);
    });

    describe('#find', function() {
      it('returns a Trello board', function() {
        trelloboardRepository.find(1).then(function(trelloboard) {
          trelloboard.cardGroups.should.be.instanceof(Array);
        });
      });
    });
  });
});
```

### The AngularJS service loader (not used when unit-testing)

```js
// trelloboard/services.js
define([
  'app', // the AngularJS kernel
  'trelloboard/repository'
], function(
  app,
  TrelloboardRepository
) {
  'use strict';

  app.service('TrelloboardRepository', TrelloboardRepository);
});
```

### Our controller

```js
define(['app', 'trelloboard/services'], function(app) {
  'use strict';

  // TrelloboardRepository is magically injected by Angular, now that we've
  // wired it up in trelloboard_services
  app.controller('TrelloboardIndexController', function($scope, TrelloboardRepository, $stateParams) {
    TrelloboardRepository.find($stateParams.id).then(function(trelloboard) {
      $scope.model = trelloboard;
    });
  });
});
```

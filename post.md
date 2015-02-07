When building a web application, one of the biggest benefits of a tool like Angular is the structure it lends to your application. While you can get started writing functional code quickly, Angular also provides a powerful system for organizing the various components of your application. Taking advantage of this organization system, known as dependency injection, will make it dramatically easier to manage your code base as it grows and your application increases in complexity. More importantly, Angular's dependency injection system makes it especially easy to test your code. 

Testing can be especially difficult in browser based applications. Testing your application involves verifying the behavior of components with very different roles. You'll need to test services that interact with remote services as well as code that directly manipulates the DOM. Angular's directives and use of dependency injection can be daunting for beginners, but becomes incredibly valuable as you begin to write unit tests against your applications.

The Angular team at Google provides two tools for testing Angular applications: [Karma](http://karma-runner.github.io), a test runner for unit testing, and [Protractor](http://angular.github.io/protractor), a test framework for writing end-to-end (E2E) tests. In this article, I'm going to be focusing on writing unit tests against Angular applications using Karma. We'll cover some unit-testing basics, Karma details, as well as Angular-specific strategies for writing unit tests that help improve overall developer productivity instead of adding an extra development burden to your application. 

## 1 Unit Testing Basics

Let's first review what unit tests are and why it's worth investing the time in writing them for web applications.

Unit tests involve running code and verifying aspects of its behavior such as function calls or return values. Tests can cover a scope as small as a single function or test the interaction between a service and its dependencies. In Angular, any code that directly verifies the behavior of other code is considered a unit test. 

Unit tests are distinct from end-to-end (E2E) tests. End-to-end testing involves using a browser automation tool like Selenium to interact with your application's interface and verify high level details like URLs, text, and HTML attributes. End-to-end tests are a useful insurance policy and sanity-check. If something is badly broken in your code, a failed end-to-end test will alert you to the need for deep investigation. 

Unit tests span a much broader range of functionality. If approached and written properly, unit tests can be a major enhancer of productivity. End-to-end tests (or worse, manually verifying new behavior) only tell you that something is wrong. Unit tests give you more direct information about the nature of failures. There's no need to adhere to a strict test-driven development (TDD) style. However, keeping your test suite relatively in sync with new code is always a good idea. 

## 2 Getting Started with Karma

Karma is a test runner for the browser supported by the Angular.js team at Google. It makes no specific assumptions about how you write your tests. At its core, Karma launches instances of the web browsers you choose, loads the files you specify, and reports the results of your tests from the browsers back to your terminal. I'll be using [Mocha](http://mochajs.org/) and [Chai](http://chaijs.com/) for this tutorial, both of which have available karma plugins for easy integration. You could just as easily choose a different tool like Jasmine. I'll be testing code in PhantomJS, a headless version of WebKit. Karma can also auto-launch Chrome, Firefox, Internet Explorer, and many other browsers. It will even instruct active browsers to rerun tests when files change.

To use Karma, you'll need to install [Node.js](//nodejs.org) if you don't have it on your system already. Note that you don't have to write your code using Node modules. It's just required for running Karma.

First, install the Karma CLI globally:

<!--code lang=bash linenums=true-->

    $ npm install -g karma-cli

Next, install Karma and the plugins we'll be using. These are installed locally in your project:

<!--code lang=bash linenums=true-->

    $ npm install --save-dev karma karma-mocha karma-chai karma-phantomjs-launcher

Karma automatically reads a `karma.conf.js` file in your project directory. Karma can automatically walk you through creating this file:


<!--code lang=bash linenums=true-->

    $ karma init

After initializing, you'll need to add `'chai'` to the `frameworks` section. Your final configuration should look like this:

<!--code lang=javascript linenums=true-->
    
    // Karma configuration
    // Generated on Mon Nov 10 2014 19:13:16 GMT-0500 (EST)

    module.exports = function(config) {
      config.set({

      // base path that will be used to resolve all patterns (eg. files, exclude)
      basePath: '',


      // frameworks to use
      // available frameworks: https://npmjs.org/browse/keyword/karma-adapter
      frameworks: ['mocha', 'chai'],


      // list of files / patterns to load in the browser
      files: [
        './src/**/*.js',
        './test/**/*.js'
      ],


      // list of files to exclude
      exclude: [
      ],


      // preprocess matching files before serving them to the browser
      // available preprocessors: https://npmjs.org/browse/keyword/karma-preprocessor
      preprocessors: {
      },


      // test results reporter to use
      // possible values: 'dots', 'progress'
      // available reporters: https://npmjs.org/browse/keyword/karma-reporter
      reporters: ['progress'],


      // web server port
      port: 9876,


      // enable / disable colors in the output (reporters and logs)
      colors: true,


      // level of logging
      // possible values: config.LOG_DISABLE || config.LOG_ERROR || config.LOG_WARN || config.LOG_INFO || config.LOG_DEBUG
      logLevel: config.LOG_INFO,


      // enable / disable watching file and executing tests whenever any file changes
      autoWatch: true,


      // start these browsers
      // available browser launchers: https://npmjs.org/browse/keyword/karma-launcher
      browsers: ['PhantomJS'],


      // Continuous Integration mode
      // if true, Karma captures browsers, runs the tests and exits
      singleRun: false
      });
    };

Now that Karma is installed and configured, we're ready to start writing code and tests. 

## 3 Basics of Dependency Injection

Before we actually dig into examples and code, I'd like to first cover dependency injection as its used in Angular. Dependency injection just means that your services specify which dependencies they expect to receive (rather than explicitly importing them). When you want to instantiate that service, an injector needs to pass the requested dependencies into the service. Normally this happens automatically in Angular. Consider the following, where we inject a child scope into a controller:

<!--code lang=javascript linenums=true-->
  
    function MyController ($scope) {
      $scope.property = 'value';
    }
    MyController.$inject = ['$scope'];

I'm using the `$inject` syntax here, but you can also let Angular try to guess the dependency names (won't work with minifed code) or pass an array to `controller` when you register:

<!--code lang=javascript linenums=true-->
  
    function MyController ($scope) {
      $scope.property = 'value';
    }
    angular.module('myApp', [])
      .controller('MyController', [
        '$scope',
      myController
    ]);

Regardless of which style you choose, what goes on under the hood is ultimately the same. You never explicitly create a new child `Scope`. Instead, Angular locates the parent scope, calls `scope.$new()` for you, and passes the result into your controller. You might also want to inject a service:

<!--code lang=javascript linenums=true-->
  
    function MyService () {
      return 'hello!';
    }
    function MyController ($scope) {
      $scope.property = 'value';
    }
    MyController.$inject = ['$scope', 'MyService'];
    angular.app('myApp', [])
    .controller('MyController', MyController)
    .factory('MyService', MyService);

Notice that we've now explicitly defined `MyService` ourselves, whereas `$scope` is created implicitly and made available by Angular's controller system. This illustrates an important distinction in dependency injection between services we explicitly provide versus `locals` that are implicitly created. Locals are most commonly used with controllers which are typically invokved using the `ngController` directive or a routing library like [ngRoute](https://docs.angularjs.org/api/ngRoute) or [AngularUI Router](https://github.com/angular-ui/ui-router). Locals will override globally available service and we will use them to easily test controllers.

##4 Using `ngMock`

Before diving in, let's review one final staple of unit testing an Angular application: the `ngMock` module. `ngMock` allows you to inject and mock Angular services for unit tests. It also decorates various core services like `$http` so you can easily write readable tests without a server. You can install `ngMock` using npm, bower, or manually. I'll be using npm to install our dependencies:

<!--code lang=bash linenums=true-->
  
    $ npm install --save angular
    $ npm install --save-dev angular-mocks

Then I'll edit the `files` array from `karma.conf.js` to include them:

<!--code lang=javascript linenums=true-->
  
    files: [
      'node_modules/angular/angular.js',
      'node_modules/angular-mocks/angular-mocks.js'.
      './src/**/*.js',
      './test/**/*.js'
    ]

Now we're ready to write code and begin testing!

## 5 Testing a Service

The easiest targets for unit testing are services, since they're just plain values that won't change depending on where we are in the application (unlike controllers which will almost always receive `locals`). First we'll need to create a new module to hold our application:

<!--code lang=javascript linenums=true-->
 
    angular.module('myApp', []);

Then we'll create a simple service:

<!--code lang=javascript linenums=true-->
  
    angular.module('myApp')
      .factory('Person', function () {
        return function Person (name) {
          this.name = name;
        };
      });

Next we'll write a test verifying that calling `new Person('Ben')` will assign a property `name` with the value `'Ben'`:

<!--code lang=javascript linenums=true-->
  
    describe('Person', function () {

      var Person;
      beforeEach(module('myApp'));
      beforeEach(inject(function (_Person_) {
        Person = _Person_;
      }));

      describe('Constructor', function () {

        it('assigns a name', function () {
          expect(new Person('Ben')).to.have.property('name', 'Ben');
        });

      });

    });

You can run `karma start` and see that our test passed. But first, let's dissect exactly what's happening here. `describe`, `beforeEach`, and `it` are provided by Mocha for organizing our tests. `module` and `inject` are provided by `ngMock`. `module` registers the `ngMock` module on our module named `'myApp'` without us needing to explicitly include `'ngMock'` in the second argument to `angular.module`. Next we call `inject` with a function that takes an argument `_Person_`. When using `ngMock`, Angular will automatically strip away leading and trailing underscores from the function we pass for the injector to invoke. Declaring `var Person` in the scope of the first `describe` block means that `Person` will always be assigned to our service before each test runs. Next we create a new Person and use `expect` (from Chai) to verify that it has the name property we expected. 

Next we'll inject another service into our `Person` to enhance its functionality. We'll be injecting a `visitor` service with a `country` property. In a real application, you might assign this information using an IP geolocation API. Right now, we're only interesting in testing how our code behaves based on the `visitor` and not how the `visitor` service itself works. We'd like to greet our `Person`, but more formally if they're from the UK:

<!--code lang=javascript linenums=true-->

    angular.module('myApp')
      .factory('Person', function (visitor) {
        return function Person (name) {
          this.name = name;
          this.greet = function () {
            if (visitor.country === 'UK') {
              console.log('Good day to you,', this.name + '.');
            }
            else {
              console.log('Hey', this.name + '!');
            }
          };
        };
      });

In order to test this in isolation, we'll manually provide a service called `visitor`:

<!--code lang=javascript linenums=true-->
  
    var Person, visitor;
    beforeEach(module('myApp'));
    beforeEach(module(function ($provide) {
      visitor = {};
      $provide.value('visitor', visitor);
    }));

[`$provide`](https://docs.angularjs.org/api/auto/service/$provide) is a service that registers other services with the injector. In this case we want to provide `visitor` as an empty object to any service that tries to inject it. 

Now we can test our greet method:

<!--code lang=javascript linenums=true-->

    describe('#greet', function () {

      it('greets UK visitors formally', function () {
        visitor.country = 'UK';
        expect(new Person('Nigel').greet()).to.equal('Good day to you, Nigel.');
      });

      it('greets others visitors informally', function () {
        expect(new Person('Ben').greet()).to.equal('Hey Ben!');
      });

    });

This is a powerful pattern that makes it easy to test individual services in isolation. `ngMock` takes a similar approach internally to decorate core services. Let's add a `create` method for sending our user's name up to a REST API and test it without a server.

<!--code lang=javascript linenums=true-->
  
    angular.module('myApp')
      .factory('Person', function (visitor, $http) {
        return function Person (name) {
          // ...
          this.create = function () {
            return $http.post('/people', this);
          };
        };
      });

First, we'll need to inject `$http`. We'll return the call to `$http.post` from `person.create` since `$http` calls return promises. Now we can test our new method:

<!--code lang=javascript linenums=true-->

    var Person, visitor, $httpBackend;
    // ...
    beforeEach(inject(function (_Person_, _$httpBackend_) {
      Person = _Person_;
      $httpBackend = _$httpBackend_;
    }));

    // ...

    describe('#create', function () {

      it('creates the person on the server', function () {
        $httpBackend
        .expectPOST('/people', {
          name: 'Ben'
        })
        .respond(200);
        var succeeded;
        new Person('Ben').create()
        .then(function () {
          succeeded = true;
        });
        $httpBackend.flush();
        expect(succeeded).to.be.true;
      });

    });

Using the mocking API provided by `ngMock` through [`$httpBackend`](https://docs.angularjs.org/api/ngMock/service/$httpBackend), we can define the request we expect to see as well as the response that will be sent. When we call `$httpBackend.flush`, Angular flushes the queue of pending requests and fulfills them with our mocked responses. 

## 6 Testing Controllers

Controllers in Angular are the glue that bind services together to directives and templates. Controllers will be the first place we look at `locals`, the services that we can provide at injection time. To test controllers, Angular exposes the `$controller` service. We can inject this service and then call it with two arguments:

1. The name of the controller as registered with `app.controller`
2. An object containing the `locals` we wish to inject

Note that `locals` only need to be provided for services that can't be injected normally. You'll almost always be passing a `$scope` which is a local service. 

First, we'll create an ultra-simple controller:

<!--code lang=javascript linenums=true-->

    angular.module('myApp')
      .controller('PersonController', function ($scope, Person) {
        this.person = $scope.person = new Person('Ben');
      });

Note that I'm setting a property `person` on both `this` and `$scope`. This isn't a pattern you would want to use in a real application, but just an easy way to illustrate how we can test both scenarios. Now we can test that both the controller and the scope that is injected into it both receive a `person` property:

<!--code lang=javascript linenums=true-->

    describe('PersonController', function () {

      var Person, controller, scope;
      beforeEach(module('myApp'));
      beforeEach(module(function ($provide) {
        $provide.value('visitor', {});
      }));
      beforeEach(inject(function (_Person_, $controller, $rootScope) {
        Person = _Person_;
        scope = $rootScope.$new();
        controller = $controller('PersonController', {
          $scope: scope
        });
      }));

      it('assigns a person to the controller', function () {
        expect(controller.person).to.be.an.instanceOf(Person);
      });

      it('assigns a person to the scope', function () {
        expect(scope.person).to.be.an.instanceOf(Person);
      });

    });

We still need to provide a mock `visitor` since our controller injects `Person` which in turn injects `visitor`. Next we inject the `$controller` service. We also inject `$rootScope` and call `$rootScope.$new` to create a new child scope. We pass this scope as a *local* to our controller and store the controller instance for tests. Then we verify that both `controller.person` and `scope.person` are instances of `Person`. Note that in our `locals` object, the keys are the names of the services that our controller is expecting. In this case, `scope` could have been an empty object and our tests would still pass. It would be easy, for example, that tested behavior that was conditional on some value inherited from a parent scope. 

## 7 Testing Directives

Directives are the most unique and powerful API in Angular's toolkit, but also one of the most difficult to fully grasp. Writing "semantic" Angular involves putting any logic that manipulates the DOM inside the compile/linking functions of a directive, and never in controllers. Breaking this contract and failing to separate logic from DOM manipulation can make testing brittle and all but impossible. However, if you're diligent about using directives to organize your DOM manipulation code well, testing is relatively painless. 

Similar to testing controllers, we'll be relying on manually triggering an Angular service that would normally be fired automatically. This time we'll be using `$compile`. Our directive will display a simple welcome message based on the output of `person.greet()`. We want to be able to pass it a specific person and see the message displayed in an `<span>` tag.

<!--code lang=javascript linenums=true-->

    angular.module('myApp')
      .directive('welcome', function () {
        return {
          restrict: 'E',
          scope: {
            person: '='
          },
          template: '<span>{{person.greet()}} Welcome to the app!</span>'
        };
    });

Notice that we can avoid actually writing any code for this directive and Angular will all the work. Now we need to test that the directive works as expected.

<!--code lang=javascript linenums=true-->
  
    describe('Welcome Directive', function () {

      var element, scope;
      beforeEach(module('myApp'));
      beforeEach(inject(function ($compile, $rootScope) {
        scope = $rootScope.$new();
        element = $compile('<welcome person="person"></welcome>')(scope);
      }));

      it('welcomes the person', function () {
        scope.person = {
          greet: function () {
            return 'Hello!';
          }
        };
        scope.$digest();
        expect(element.find('span').text()).to.equal('Hello! Welcome to the app!');
      });

    });

In our `beforeEach` function, we inject `$compile`, the service that Angular automatically uses internally when running your app. We create a new `scope` and call `$compile` on the HTML text we want to compile. That returns a function that accepts a scope to bind to. In this case, we're using an isolate scope. Then, in our test, we actually populate it with a person. Instead of actually using an instance of `Person`, we just create a mock object that will behave the way a `person` would behave. Next, we need to call `scope.$digest`. Angular normally does this automatically when running our application. However, under test, we want to have granular control over digest cycle so we can control how our updates are applied. To illustrate this, add the following line above the `$digest` call:

<!--code lang=javascript linenums=true-->

    expect(element.find('span').text()).to.equal('{{person.greet()}} Welcome to the app!');

You'll notice that the template has been populated already (that happens in the compile stage). However, because a digest cycle has not run yet, the expression has not yet been replaced by its evaluated result.

Now let's add some code to the directive's linking function to make the text color change based on the person's favorite color when hovering:

<!--code lang=javascript linenums=true-->
  
    angular.module('myApp')
      .directive('welcome', function () {
        return {
          restrict: 'E',
          scope: {
            person: '='
          },
          template: '<span>{{person.greet()}} Welcome to the app!</span>',
          link: function (scope, element) {
            var original = element.css('color');
            element.on('mouseenter', function () {
              element.css('color', scope.person.favoriteColor);
            });
            element.on('mouseleave', function () {
              element.css('color', original);
            });
          }
        };
      });

We've added a `link` function that stores the element's original color value, changes it to the person's favorite color on `mouseenter`, and reverts it on `mouseleave`.

We'll also add a test against this new behavior:

<!--code lang=javascript linenums=true-->

    it('displays the person\'s favorite color on hover', function () {
      scope.person = {
        greet: function () {
          return 'Hello!';
        },
        favoriteColor: 'blue'
      };
      scope.$digest();
      element.triggerHandler('mouseenter');
      expect(element.css('color')).to.equal('blue');
      element.triggerHandler('mouseleave');
      expect(element.css('color')).to.be.empty;
    });

Note that we only need to manually kick off a digest in our tests when we change the scope. Since we're dealing with the DOM on both sides (DOM events triggering DOM changes) there's no need to check the `scope` for changes.

## 8 Advanced Karma

While we've been using a very small subset of Karma's functionality, its plugin API offers many ways to use it with more advanced workflows:

* [karma-bro](http://github.com/nikku/karma-bro) Test Browserify projects with Karma
* [karma-coffee-preprocessor](https://github.com/karma-runner/karma-coffee-preprocessor) Test CoffeeScript
* [karma-sauce-launcher](https://github.com/karma-runner/karma-sauce-launcher) Launch 200+ browser and OS combinations and run your tests
* [karma-coverage](https://github.com/karma-runner/karma-coverage) Generate a code coverage report using Istanbul

Karma even has an [API](http://karma-runner.github.io/0.12/dev/public-api.html) that lets you easily launch Karma servers from code. While Karma is supported by the Angular team at Google, it can be used to run tests for any JavaScript browser-based application. 

We're going to tackle adding test coverage here to make sure our unit tests are covering all the different possible paths our code can take. 

You'll change a few keys in your `karma.conf.js` file:

<!--code lang=javascript linenums=true-->

    preprocessors: {
      './src/**/*.js': ['coverage']
    },
    reporters: ['progress', 'coverage'],
    coverageReporter: {
      type : 'html',
      dir : 'coverage/'
    }

Now you can rerun Karma and see that it reports 100% coverage!

## 9 Conclusion

While some of Angular's verbose conventions can be difficult to grasp at first, I hope I've illustrated how they can help you be more productive when building applications. Although it's tempting to see time spent unit testing as productive time lost, you're setting yourself up for long term success by testing. As your application grows in size and scope, maintaining strong test coverage will make it easier to split up your functionality and trust that each piece will function as specified. 

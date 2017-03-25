# message-board

## Table of Contents

  1. [Naming Convention](#naming-convention)
  1. [Modules](#modules)
  1. [Routes](#routes)
  1. [Controllers](#controllers)
  1. [Services](#services)
  1. [Directives](#directives)
  1. [Comments](#comment-standards)
  1. [Test](#test) TODO

## Naming Convention

- **File names**   
  Dasherized with no upper cases 

  - **Controller file name**   
    Postfix with '-ctrl.js'. e.g.  users-ctrl.js
  
  - **Test file name**  
    Postfix with '-spec.js'. e.g., 'ks-click-dialog-spec.js'

- **Module name**   
  LowerCamelCase. e.g. 'ks'

- **Controller name**  
  UpperCase ending with 'Ctrl', e.g. 'UsersCtrl'

- **Directive name**   
  LowerCamelCase starting with 'ks'. e.g. 'ksClickDialog'

- **Utility service name**  
  LowerCamelCase starting with 'ks'. e.g. 'ksFlashMessage'

- **Data service name**  
  UpperCamelCase e.g. 'User', 'CommonData'
  
## Routes

- **Use resolve to inject data**

  *Why?* With resolved data, the page is only rendered when all the data is available.
  Without resolving data, the user may see a broken page without data ready. 
  You should avoid having the user see any empty views whilst the data is loading.

        // --- Avoid ---
        // my-ctrl.js
        var MyCtrl = function($scope, $stateParams, Customer) {
          Customer.get($stateParams.id).then(function (response) {
            $scope.customers = response;
          });
        }
        //routing.js
        $stateProvider.state('customers.show', {
          url: '/customers/:id',
          template: template,
          controller: 'MyCtrl'
        });
        
        // --- Recommended ---
        // my-ctrl.js
        var MyCtrl = function($scope, customers) {
          $scope.customers = customers;
        }

        //routing.js
        $stateProvider.state('customers.show', {
          url: '/customers/:id',
          template: template,
          controller: 'MyCtrl',
          resolve: {
            customers: ['Customer', '$stateParams', function(Customer, $stateParams) {
              return Customer.get($stateParams.id);
            }];
          }
        });

   Ideally, we should not couple resolve logic with the router itself.  Instead, reference a `resolve` function of a Controller in the router.
   This keeps resolve dependencies inside the same file as the controller and the router free from logic.

## Modules

- **Definitions**

  Declare modules without a variable using the setter and getter syntax.

        //avoid
        var app = angular.module('ks', []);
        app.controller('MyCtrl', ... );
        app.factory('MyData', ... );

        //recommended
        angular.module('ks', []);
        angular.module('ks').controller('MyCtrl', ...);
        angular.module('ks').factory('MyFactory', ...);

- **Methods**

  Pass functions into module methods rather than assigning them as callbacks.

        // avoid
        angular.module('app').service('SomeService', function SomeService () {
          ...
        });

        // recommended
        function SomeService () {
          ...
        }
        angular.module('app').service('SomeService', SomeService);

  This aids with readability and reduces the volume of code "wrapped" inside the Angular framework.

- **IIFE scoping**

  To avoid polluting the global scope with our function declarations that get passed into Angular, ensure build tasks wrap the concatenated files inside an IIFE.

        (function () {
          'use strict';
        
          angular.module('app', []);
          
          function SomeService () { 
          }
          
          angular.module('app').service('SomeService', SomeService);
            
        })();

**[Back to top](#table-of-contents)**

## Controllers

- **Only instantiate controllers through routes or directives.**

  *Why?* It allows reuse of controllers and encourages component encapsulation.
  Also, do not define a controller to scope a section of html if not used in routing.

        // avoid
        <div ng-controller="MyCtrl">
          <h1>{{title}}</h1>
          <section>My Title</section>
        </div>

        // recommended
        <h1>{{title}}</h1>
        <section>My Title</section>

- **Presentational logic only in a controller**

  *Why?* The Controller must be kept as simple as possible by only dealing with view model data, 
  making the controller act as a ViewModel and controlling the data flowing between the Model and the View Presentation. Also any business logic in the controller makes testing more difficult.

        // avoid
        var MainCtrl = function($scope) {
          ...
          $http.get('/users').success(function (response) {
            $scope.users = response;
          });
          ...
        }

        // recommended
        var MainCtrl = function($scope, User) {
          ...
          User.getUsers().then(function (response) {
            $scope.users = response;
          });
          ...
        }

- **DOM manipulation? NO.**

 Do NOT read or write to a DOM element in the controller. This only happens in a directive.

- **Data Conversion? NO.**

 Do NOT manipulate data within the controller -- use a filter instead.


        // avoid
        var MainCtrl = function($scope, User) {
          ...
          User.getUsers().then(function(response) {
            for (var key in response) {
              ... filter out response with some complex logic
            } 
            $scope.users = response;
          });
          ...
        }
        
        // recommended
        var MainCtrl = function($scope, User, ksCustomFilter) {
          ...
          User.getUsers().then(function (response) {
            $scope.users = ksCustomFilter(response);
          });
          ...
        }


**[Back to top](#table-of-contents)**

## Services

- **Use a `factory` instead of a `service`**

  Since all Angular services are singletons, there in only one instance of a given service per injector.
  Which means, although a service is instantiated with the `new` keyword, there would be only one instance that is used
  unless `new` is intentionally invoked to create a new instance.

  Treat a service object like a class with static methods and avoid exporting a service as a single function, because it's easier to see the definition of service within the return black.

        // avoid
        function logger() {
          var logFunc = function(msg) {
            ...
          };
          this.logError = logFunc
        }
        angular.module('app').service('logger', logger);
        
        // recommended
        function logger() {
          var logFunc = function(msg) {
            ...
          };
          return {
            logError: logFunc
          }
        }
        angular.module('app').factory('logger', logger);

- **Single Responsibility**

  A Factory should have a single responsibility that is encapsulated by its context.
  Once a factory begins to exceed that singular purpose, a new factory should be created.

- **No DOM manipulation**

  Do NOT read or write to DOM elements in the controller. This only happens in a directive.

- **Avoid Data Conversion**

  Try to avoid it and use a filter instead, if common.

        // avoid
        function logger() {
          var logFunc = function(input) {
            for(var key in hash) {
              ... filter input with some complex logic
            }
          };
          return {
            logError: logFunc
          } 
        }
        
        // recommended
        function logger(ksCustomFilter) {
          var logFunc = function(input) {
            var arr = ksCustomFilterFilter(input);
          };
          return {
            logError: logFunc
          } 
        }

- **Use the simple `$http` instead of the complex `$resource` for data service.**

  Unlike `$http`, a resource does not return `promise`, which makes coding more difficult.
  `$resource` uses one base url (e.g. `/users/:id`), thus any custom urls need to be redefined.
  And when we redefine a custom url we need to follow the resource way, which is not very conventional.

  There is also no `PUT` method in a resource, you need to define it separately anyway.

  Resource also uses static and instance methods together, which makes it more confusing than just `$http` method.
  Simply because it's implicit, not explicit, it requires more time to debug and find the proper usage.

        // avoid
        app.factory('Notes', function($resource) {
          return $resource('/notes/:id', null, {
            update: { method:'PUT' },
            autocomplete: {url: '/notes/autocomplete', method:'GET'},
            addComment: ????
            removeComment: ????
          });
        });

        // recommended
        app.factory('Notes', function($http) {
          return {
            get: function(id, params) {
              return $http.get('/notes/'+id);
            },
            save: function(id, data) {
              var url = id ? '/users/'+id : '/users';
              return $http.get({url: url, method: (id ? 'PUT' :'POST'), data: data});
            },
            autocomplete: function(params) {
              return $http.get('/notes/autocomplete.json', params: params});
            }
          }
        });

- **Return a promise instead of a result from an `$http` call.**

  *Why?* A service user can chain the promises together and take further action after the data call 
  completes and resolves or rejects the promose.

        // avoid
        app.factory('Avenger', function($q, $http) {
          var deferred = $q.defer();
          $http.get('/avengers').success(function(result){
            deferred.resolve(result); 
          });
          return deferred.promise;
        }); 
        
        app.controller('MyCtrl'), function($q, Avengers) {
          Avengers.then(function(data) {
            //... do something 
          })
        })
        
        // recommended
        app.factory('Avenger', function($http) {
          return $http.get('/avengers');
        }); 
        
        app.controller('MyCtrl'), function(Avengers, Avengers2) {
          Avengers.then(function() {
            //... do something 
          })
        })

**[Back to top](#table-of-contents)**

## Directives

- **Prefix it with application name** (e.g. `ks-`). 

  *Why?* It is easy to know it's made by KineticSocial with source code in `js/directives` directory. 
  It also differentiates names from external directives. e.g. `ng-dialog`, `auto-complete`, etc.

        <!-- Avoid -->
        <div alert></div>

        <!-- Recommended -->
        <div ks-alert></div>

- **Use isolated scope directives for elements. Use shared scope directives for attributes.**

  *Why?* Using an isolated scope forces you to expose an API by giving the component all the data it needs.
  This increases reusability and testability. 

  Are you defining a common behaviour or a unique behaviour?
  If the answer is "a unique behaviour", use an isolated scope. If common, do not isolate the scope.

        <!-- Avoid -->
        <div ks-header template="here.html"></div>

        <!-- Recommended -->
        <ks-header template="here.html"></ks-header>

  When using a shared scope directive, you should not rely on any existing data. 
  This injects the behaviour to the current scope.

        <!-- Avoid -->
        <ks-alert="This Message"></ks-alert>

        <!-- Recommended -->
        <div ks-alert="This Message></div>

- **Use only `E`, `A` for restriction**, or omit it as default to `EA`.
  
  *Why?* Class name declarations are confusing with real class names.
  Comment directives are not working well with older versions of IE, and they're not safe with HTML minification. 


- **Pro-Patterns**

  - Keep page refresh in mind, and do not lose current state with F5
  - Use `$timeout` instead of `setTimeout`
  - Use `$window` instead of `window`
  - Use `$document` instead of `document`
  - Use `$http` instead of `$.ajax`
  - Use `$interval` instead of `setInterval`
  - Use `$q` instead of callbacks 

  *Why?* This makes tests easier and faster to run. Also you don't need to run `$digest` or `$apply` separately.

- **Anti-Patterns**

  - Don't use any global variables.
    Use dependency injection instead. It makes testing and refactoring easier.

  - Don't use `$` as a function or variable name.
    So that we can easily differentiate Angular core function name or variables.

  - Don't use jQuery selectors. Find elements within `element` scope. 
    Use `querySelector` instead. e.g. `element[0].querySelector('div')`.

  - Don't use jQuery to generate templates or DOM. 
    Use directive templates instead.

  - Don't prefix directive names with x-, polymer-, ng-. 
    They may conflict with future native directives.

  - Avoid template-only directives. 
    Use `ng-include` instead.

  - Do not use `ng-init`. 
    Use `ks-value` or controller instead.

  - Do not use `ng-controller`. 
    Use directive or route instead.

  - Do not use `$watch` in a controller.
    Use directive or event-driven function instead.

  - Do not use jQuery in a directive.
    Use `querySelector` or `element` instead.

**[Back to top](#table-of-contents)**

## Comments

- **jsDoc**

  Use [jsDoc syntax](http://usejsdoc.org/) for documentation; names, descriptions, params and returns.
  Also use the `ngdoc` tag to group it. e.g., service, directive, filter, etc.
  so that it can be properly grouped by [angular-jsdoc](https://github.com/allenhwkim/angular-jsdoc)

        /**
         * @ngdoc service
         * @name SomeService
         * @desc Main application Controller
         */
        function SomeService (SomeService) {

          /**
           * @memberof SomeService
           * @desc Does something awesome
           * @param {Number} x - First number to do something with
           * @param {Number} y - Second number to do something with
           * @returns {Number}
           */
          this.doSomething = function (x, y) {
            return x * y;
          };
        }

**[Back to top](#table-of-contents)**

## JSHint

- Hint file is at `.jshintrc`

- **Install**

  - **For vim users**
    
    - $ npm install jshint -g
    - Install Plugin https://github.com/wookiehangover/jshint.vim

  - **For SublimeText users**

    - Ctrl+Shift+P or Cmd+Shift+P in Linux/Windows/OS X
    - type install, select Package Control: Install Package
    - type js gutter, select JSHint Gutter

  - **For Atom users**

    - $ apm install atom-jshint
    - Reference: https://atom.io/packages/atom-jshint

## Test
- Postfix file name with `-spec.js` in the directory that file is in, not in a `test` or `spec` directory.
- TODO

**[Back to top](#table-of-contents)**

## References

- [Todd Motto’s styleguide](https://github.com/toddmotto/angularjs-styleguide)
- [John Papa’s styleguide](https://github.com/johnpapa/angularjs-styleguide)
- [Minko Gechev’s styleguide](https://github.com/mgechev/angularjs-style-guide)
- [Google AngularJS and Closure styleguide](http://google-styleguide.googlecode.com/svn/trunk/angularjs-google-style.html)
- [Python PEP 8 Styleguide](http://legacy.python.org/dev/peps/pep-0008/)


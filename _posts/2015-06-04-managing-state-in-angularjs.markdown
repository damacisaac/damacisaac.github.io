---
layout: post
title:  "Managing State In AngularJS"
date:   2015-06-04 21:10:00
categories: angularjs
---

A common problem that developers will face when building an Angular application is how to manage state while also presenting it in views. The issue is that while our view template uses `$scope` and `$rootScope` for data binding, we normally want to manage our state inside services, which don't have access to `$scope`. We're then left to decide what the best way is to communicate between controllers and services so that we can continue to manage state in our services while still being able to present it with `$scope`. There are a couple of solutions to this problem, some better than others. Here are my thoughts:

**Solution 1: Pass `$scope` to the service.**

{% highlight ruby %}
(function () {

    // @ngInject
    function UserCtrl($scope, Session) {
        Session.setCurrentUser($scope);
    }

    angular.module('app')
        .controller('UserCtrl', UserCtrl);
})();
{% endhighlight %}

This is **not** a good solution. Our services should not depend on `$scope`.

**Solution 2: Use `$scope.$watch`.**

{% highlight ruby %}
(function () {

    // @ngInject
    function UserCtrl($scope, Session) {
        $scope.$watch(function () {
            return Session.getCurrentUser();
        }, function (newVal, oldVal) {
            if ( newValue !== oldValue ) {
                $scope.currentUser = newVal;
            }
        });
    }

    angular.module('app')
        .controller('UserCtrl', UserCtrl);
})();
{% endhighlight %}

Here we query the value of `currentUser` inside a `$watch`. The callback function will be called if the value returned by `Session.getCurrentUser()` changes. We're now sure that `$scope.currentUser` will always be updated.

Although it works, it's not a great solution. In my own applications, I will do everything I can to eliminate `$watch`s. They're inefficient and they're ugly. The use of `$watch` itself means that we've copied state somewhere and are trying to keep multiple instances of it synced. We should avoid this if possible.

**Solution 3: Use `$scope.$on`.**

{% highlight ruby %}
(function () {

    // @ngInject
    function UserCtrl($scope, Session) {
        $scope.currentUser = Session.getCurrentUser();

        $scope.$on('user-updated' function(event, data) {
            $scope.currentUser = data;
        });
    }

    angular.module('app')
        .controller('UserCtrl', UserCtrl);
})();
{% endhighlight %}

This is another solution that's often proposed, but again it falls short. If we stick to this method, we have to remember to add the `$scope.$on` method every time we query for `currentUser`. We also have to make sure to `$scope.$emit` any time we update `currentUser`. These are both error prone.

There's also a more conceptual problem with using events to manage state. Events should be reserved for more *event-like* things than changing data. If a user updates his phone number, should this really be represented as an event in our application? I would say no.


**Solution 4: Preserve Reference with `angular.extend()`**.

{% highlight ruby %}
(function () {

    // @ngInject
    function Session(CurrentUser)
        this.currentUser = {};

        CurrentUser.query()
            .then(function (response) {
                angular.extend(this.currentUser, response);
            });
    }

    // @ngInject
    function UserCtrl($scope, Session) {
        $scope.currentUser = Session.currentUser;
    }

    angular.module('app')
        .service("Session", Session)
        .controller('UserCtrl', UserCtrl);
})();
{% endhighlight %}

In this example, the `Session` service defines an empty `this.currentUser`, and then makes an API call to obtain the data from the server. Once the call returns, we `angular.extend` the empty `this.currentUser` object with the data. This preserves the reference to `this.currentUser` so that all controllers and services using the reference are updated as well.

While this works fine, and involves a minimal amount of code in the controller, there is a tiny problem. We now have to be careful to not override the reference to `this.currentUser`. If, somewhere in our application, we write `Session.currentUser = newUser`, any controller or service using the old reference will fall out of sync.

Another problem with `angular.extend` is that it doesn't *remove* data from an object. If at some point, we want to perform an update that removes some key-values from `this.currentUser`, using `angular.extend` wont be enough.

**Solution 5: Wrap State in a Closure**.
{% highlight ruby %}
(function () {

    // @ngInject
    function Session(CurrentUser)
        this.currentUser = {};

        CurrentUser.query()
            .then(function (response) {
                angular.extend(currentUser, response);
            });

        this.getCurrentUser = function() {
            return this.currentUser;
        };
    }

    // @ngInject
    function UserCtrl($scope, Session) {
        $scope.getCurrentUser = Session.getCurrentUser;
    }

    angular.module('app')
        .service("Session", Session)
        .controller('UserCtrl', UserCtrl);
})();
{% endhighlight %}

This is the final solution. Here, instead of binding the value of `currentUser` directly to `$scope`, we instead bind a getter. Now, if we need the value of `currentUser` in our controller or in a template, we can just call `$scope.getCurrentUser()`. We also don't have to worry about losing the reference when `currentUser` is updated!

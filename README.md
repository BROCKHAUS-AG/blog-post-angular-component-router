# Experiences on using the new ngComponentRouter in Angular 1.5
While working on an angular project for one of our clients, Fabian Raetz and me had the pleasure to try out the new ngComponentRouter and see if it fits our needs.
We were especially interested in the capability to reuse existing controller instances and how to get some [$resolve](https://docs.angularjs.org/api/ngRoute/provider/$routeProvider#when)-like functionality from ngRoute in the new ngComponentRouter.
In this blog post you will learn about our experiences.

## Introduction
We use iFrames to isolate different parts of our application into independent websites. We have an application [component](https://docs.angularjs.org/guide/component) which is responsible for loading the correct website and pass the remaining url parts to the iFrame. Therefore our url contains the name of the website and the location path for it. We synchronize the location of the iFrame with the outer website to support bookmarking and deep linking.

```
/apps/:app/:location*
```

The problem with the existing ngRoute is that it recreates the application controller/view every time the url changes. No matter if the app stays the same or not. The only possibility to achieve something similar to this is to use the query string to store the website's location and set `reloadOnSearch` to false. This approach has some drawbacks and does not meet our requirements.

While searching for alternatives and figuring out that [ui-router](https://github.com/angular-ui/ui-router) doesn't support this either, we discovered the new [new router design docs](https://docs.google.com/document/d/1I3UC0RrgCh9CKrLxeE4sxwmNSBl3oSXQGt9g3KZnTJI) which contain the following two statements which made us quite curious.

> ###### A way to preserve state of certain views
>
> ngRoute 1.0 always destroys/recreates controllers/views when navigating between routes

and

> **Note**: Although the target for this is Angular 2.0, it might be possible to backport this new router to Angular 1.x.

After noticing that the new ngComponentRouter has moved from its [home](https://github.com/angular/router) to the [angular2 codebase](https://github.com/angular/angular) and can be compiled for Angular 1.x we decided to give it a try.

## How to get the ngComponentRouter up and running
As the Angular team recently decided to finally package the new ngComponentRouter you can install it from npm (`npm i @angular/router`). As an alternative you could take the compiled version from [Pete Bacon Darwin's](https://github.com/petebacondarwin/ng1-component-router-demo) or [Brandon Robert's](https://github.com/brandonroberts/angularjs-component-router) demo projects which helped us a lot to get started.

**NOTE**: The Angular team [fixed](https://github.com/angular/angular/commit/6f1ef33e320547f6c68867fa28d1189be7fa3519) [some](https://github.com/angular/angular/commit/e73fee715668740f1579093f61fea0f08d44da18) [bugs](https://github.com/angular/angular/commit/0f22dce036bd5cb3242edafb119256a6433dd4f4) in the Angular 1.x integration layer lately and we are optimistic that the router reached an almost production-ready state.

### Compiling it ourself
The new router resides inside the [Angular 2 codebase](https://github.com/angular/angular/tree/master/modules/angular1_router) since it is used by both Angular 1.x and Angular 2. In case you encounter some bugs and want to report or even fix them, you need to file / fix them in the [Angular 2 Repository](https://github.com/angular/angular).

To build the Angular 1.x router you can use the following commands:
```bash
git clone https://github.com/angular/angular.git
cd angular
npm install

npm run build
gulp buildRouter.dev
cp dist/angular_1_router.js to/your/path
```

**NOTE**: We encountered [some issues](https://github.com/angular/angular/issues/7457) with the build system while working on Windows. For now we would recommend you to use some UNIX-like OS for compiling.

### Creating our first routes
Since we are targeting Angular 1.5 we took the chance to also evaluate the new component syntax, especially in combination with the new routing. For an introduction to the component syntax see the [Angular Developer Guide](https://docs.angularjs.org/guide/component).

To fully understand the examples and why we need to do some things the way we do them, is that the new ngComponentRouter is using hierachical routes. In a perfect angular 1.5 app every route is bound to a component which can in turn register new sub routes. You can find more details in the new [Component Router Developer Guide](https://docs.angularjs.org/guide/component-router).

```js
var portalModule = angular.module('portal', ['ngComponentRouter']);
portalModule.value('$routerRootComponent', 'portal');
portalModule.component('portal', {
  templateUrl: 'components/portal.html',
  $routeConfig: [
    { path: '/home', component: 'home', name: 'Home', useAsDefault: true },
    { path: '/apps/:name/:location*', component: 'app', name: 'Apps' },
  ],
});
```

As you can see in our first example, we are setting up a `$routerRootComponent` for the router to search for an initial route configuration which is also the [suggested solution](https://docs.angularjs.org/guide/component-router#root-router-and-component). As an alternative you could provide the router with an initial configuration by using a run block which Pete Bacon Darwin is doing in [his sample project](https://github.com/petebacondarwin/ng1-component-router-demo/blob/master/app/app.js).

Our portal component defines two **route definitions**. The first route defines the default route which will load a component called `home`. This is achieved by setting `useAsDefault` to `true`.

To load some part of a specific application into the iFrame, our app component needs to know the corresponding `name` and `location`. This is achieved by defining two **route parameters**. As you might have already noticed we are using a [star path segment](https://github.com/angular/angular/blob/75343eb34007579be9cdc803da834c38e02ae12c/modules/angular2/src/router/rules/route_paths/param_route_path.ts#L69-L81) for the `location` parameter which catches the remaining parts of the url (including following slashes `/`). This is similar to the [ngRoute star path](https://docs.angularjs.org/api/ngRoute/provider/$routeProvider#when).

## Router Lifecycle Hooks
[Router Lifecycle Hooks](https://docs.angularjs.org/guide/component-router#router-lifecycle-hooks) are a new concept of the ngComponentRouter which give you fine grained access to when and if components can be activated, deactivated and reused. This is achieved by implementing several of the following methods `$routerCanReuse`, `$routerOnActivate`, `$routerOnReuse`, `routerCanDeactivate` and `$routerOnDeactivate` in the component's controller. The last hook (`$routerCanActivate`) must be implemented in the component definition so that the controller may not be instantiated.

We want our component to reuse the controller (and the view) as long as the application's `name` doesn't change. This will prevent the view and its iFrame to be recreated on every `location` change.

```js
portalModule.component('app', {
  controller: function () {
    this.$routerCanReuse = function (next, prev) {
      return next.params.name === prev.params.name;
    };

    this.$routerOnActivate = (next) => {
      this.loadAppIntoIFrame(next.params.name, next.params.location);
    };

    this.$routerOnReuse = (next, prev) => {
      if (this.hasAppLocationChanged(next, prev)) {
        this.changeAppLocation(next.params.location);
      }
    };

    this.hasAppLocationChanged = (next, prev) => {
      ...
    };
    
    this.changeAppLocation = (appLocation) => {
      ...
    };
    
    this.loadAppIntoIFrame = (appName, appLocation) => {
      ... // among others set this.iframeSrc
    };
  },
  template: `<iframe ng-src="{{$ctrl.iframeSrc | trustAsResourceUrl}}"></iframe>`,
});
```

To fulfill our needs, we implemented several lifecycle hooks. `$routerOnReuse` is called every time the route changes. Depending on whether or not the component should be reused, `$routerOnReuse` or `$routerOnActivate` are called.

`$routerOnActivate` will look up the application url. The `name` of all applications are stored in a dictionary which map to the corresponding url and full name. Afterwards the url is stored in `iframeSrc` which is bound to the iFrames src. To set ng-src you have to use the [$sce service](https://docs.angularjs.org/api/ng/service/$sce) to [make a trustworthy resource url](https://stackoverflow.com/questions/24163152/angularjs-ng-src-inside-of-iframe#answer-24163343).

`$routerOnReuse` will send the new location (if it changed) to the iFrame via [postMessage-API](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). We tried to update the iFrame's source using ng-src but the new location wasn't always visible within the iFrame.

## How to get a $resolve-like behaviour
In some parts of our application we use ngRoute's `$resolve` functionality to load external resources before navigating to the new view. This is done to ensure that all data is present when the view is rendered. Another benefit is that you can run different checks (e.g. permissions) before the controller is instantiated / view is shown.

The new ngComponentRouter does not provide `$resolve` anymore. Instead you can use `$routerCanActivate` and `$routerOnActivate` to load external resources / perform additional checks. If some of your checks fail you can return a rejected promise - like it can be done in `$resolve` - to prevent the actual navigation from happening.

```js
portalModule.component('accountDetails', {
  controller: function ($http, $rootRouter) {
    this.$routerOnActivate = (next) => {
      return $http.get('http://.../api/accountDetails/' + next.params.id).then(result => {
        this.accountDetails = result.data;
      }, error => {
        if (error.status === 404) {
          $rootRouter.navigateByInstruction(['NotFound']);
        } else { 
          // prevent navigation from happening
          throw error;
        };
      });
    };
  },
  templateUrl: 'components/account-details.html',
});
```

In our example above we use `$routerOnActivate` to fetch some accountDetails with the `$http`-service and store them on the controller instance if everything went well. If the promise is rejected with a 404 status code, we instruct the router to redirect to the `NotFound` route. In all other cases we cancel the navigation by rejecting the promise. Beware that this can lead to an empty page if the user reloads the page.

We use `$routerOnActivate` in favor of `$routerCanActivate` since we didn't find a way to pass data from the `$routerCanActivate` method to the controller instance. Another advantage is that we can specify `$routerOnActivate` as part of the controller whereas `$routerCanActivate` needs to be defined on the component definition (one thing that bugged us about the `$resolve`-approach too).

## Conclusion (tl;dr)
It took us some time to get startet with the new ngComponentRouter. First of all we had to compile the router ourselves which - even today - [doesn't](https://github.com/angular/angular/issues/7457) really [work on windows](https://github.com/angular/angular/issues/7457). Since the angular team now provides a npm package you can easily install it via `npm install @angular/router`. While figuring out how the ngComponentRouter ought to work, we found some bugs and sent a few pull requests which are mostly related to the angular-1 integration layer. Thanks to the angular team and especially to [Pete Bacon Darwin](https://github.com/petebacondarwin), who takes care of the angular 1.x ngComponentRouter lately, all our PR's got merged by now and are integrated into the official npm package.

After our issues were solved, we were quite happy with the new component router. It fulfills our requirements and is yet easy to use. With the help of the new router lifecycle methods we were able to gain fine grained control over when and how components are reused. We could even improve our old $resolve fetching logic by inlining it into the controller.

Another great thing about the new router is that is works great with components. Our thoughts about how to use and isolate directives and the new component concept aligned quite well so we took the chance to start using them today.

Thx for reading. If you have any questions or comments feel free to ask ;)

P.S.: If you are interested in how and especially why we are separating our application into several smaller applications and integrate them via iFrames, read our upcoming blog post ;)

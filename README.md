# Experiences on using the new ngComponentRouter in Angular 1.5
While working on an angular project for one of our clients, Fabian Raetz and me had the pleasure to try out the new ngComponentRouter and see if it fits our needs.
We were especially interested in the capability to reuse existing controller instances and how to get some $resolve-like functionality from ngRoute in the new ngComponentRouter.
In this blog post you will learn about our experiences.

## Introduction
In our application we use iFrames to seperate different parts of our application into independent websites. We have an application component which is responsible for loading the correct website and pass the remaining url parts to the iFrame. Therefore our url contains the name of the website and the location path for it.

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


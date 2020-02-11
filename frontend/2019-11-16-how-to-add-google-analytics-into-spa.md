# Integrating Google Analytics With Angular 2+

Source: [https://scotch.io/](https://scotch.io/tutorials/integrating-google-analytics-with-angular-2)

---

You now have an Angular app with two components that you can navigate easily between. It is now time to add our Google Analytics code to track the user activity. We do this by adding the tracking code into our `index.html` file. However, since we are building a single page application, we must send our page views manually and therefore, we remove the `ga('send', 'pageview');`line, which is responsible for transmitting the page views, from our code.

```html

<script>
    (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
        (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
        m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
    })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

    ga('create', 'UA-xxxxx-xxx', 'auto');  // Change the UA-ID to the one you got from Google Analytics

</script>

```

To manually send our page views to Google Analytics, we import the `Router` and `NavigationEnd` from `@angular/router` into our `app.component.ts`. We then subscribe to the `router.events` property to trigger a page view when we move from one route to another.

```typescript

import { Component } from '@angular/core';
import {Router, NavigationEnd} from '@angular/router'; // import Router and NavigationEnd

    // declare ga as a function to set and sent the events
    declare let ga: Function;

@Component({
    selector: 'app-root',
    templateUrl: './app.component.html',
    styleUrls: ['./app.component.css']
})
export class AppComponent {

    title = 'app';

    constructor(public router: Router) {

    // subscribe to router events and send page views to Google Analytics
    this.router.events.subscribe(event => {

        if (event instanceof NavigationEnd) {
        ga('set', 'page', event.urlAfterRedirects);
        ga('send', 'pageview');

        }

    });
    }

}

```

Now we need to create links to navigate between our components. Add the following router link to your `component-1.component.html` file to navigate to `component-2`.

```html

<p>
    component-1 works!
</p>

<button type="button" class="btn btn-primary" routerLink="/component-2">Go To Component 2</button>

```

Add another router link to `component-2.component.html` to navigate back to `component-1`.

```html

<button type="button" class="btn btn-primary" routerLink="/">Go To Component 1</button>

```

Now serve your application, go to [http://localhost:4200](http://localhost:4200/) and move from`component-1` to `component-2`.

Congratulations, you can now track the different pages a user visits when he accesses your website.


## Add Event Tracking

We may want to do more than just track a users page visits on a site. Using a real world example of a picture sharing app, we may want to track when a user clicks the like button on a particular photo and why. We do this by creating a service with an event emitter that takes in the `eventCategory`, `eventAction`, `eventLabel` as well as `eventValue` and submits it to Google Analytics.

```typescript

import { Injectable } from '@angular/core';

declare let ga:Function; // Declare ga as a function

@Injectable()
export class GoogleAnalyticsService {

    constructor() { }


    //create our event emitter to send our data to Google Analytics
    public eventEmitter(eventCategory: string,
                    eventAction: string,
                    eventLabel: string = null,
                    eventValue: number = null) {
        ga('send', 'event', {
            eventCategory: eventCategory,
            eventLabel: eventLabel,
            eventAction: eventAction,
            eventValue: eventValue
        });

    }

}

```

We then import our service to the `app.module.ts` file and add it as a provider.

```typescript

import {GoogleAnalyticsService} from "./google-analytics.service"; // import our Google Analytics service

providers: [GoogleAnalyticsService], //add it as a provider

```

Assuming that our component-2 is a page where a user has shared a picture, we add an image and a like button to the `component-2.component.html`. Our `like` button, will have a click event that triggers the `sendLikeEvent()` function.

```html

<div class = "row" style="padding-top:150px;">
    <div class = "col-md-5 offset-3 text-center">

    <img src="assets/man-1352025_960_720.png" width="500px" height="300px" style="padding-bottom:30px;"><br/>
    <button type="button" class="btn btn-primary btn-lg center" (click)="SendLikeEvent()">Like</button>

    </div>
</div>

```

In the `component-2.component.ts file`, we import our `GoogleAnalyticsService` and add it to our `sendLikeEvent()` function. This will pass our `eventCategory, eventAction, eventLabel` and `eventValue` details to our service which will submit them to Google Analytics.

```typescript

import { Component, OnInit } from '@angular/core';
import {GoogleAnalyticsService} from "../google-analytics.service"; // import our analytics service

@Component({
    selector: 'app-component-2',
    templateUrl: './component-2.component.html',
    styleUrls: ['./component-2.component.css']
})
export class Component2Component implements OnInit {

    constructor(public googleAnalyticsService: GoogleAnalyticsService) { }

    ngOnInit() {
    }

    SendLikeEvent() {
    //We call the event emmiter function from our service and pass in the details
    this.googleAnalyticsService.eventEmitter("userPage", "like", "userLabel", 1);
    }

}

```

Now go to your browser and click the like button on your component-2 page.

Check your Google Analytics Dashboard’s real time event tracker and you will see the values we passed in from our component-2.

## Possible issues

If resource cannot be loaded due to http/https security issues or/and cross-script protect, you have to add this references into meta like this:

```html

<head>
<meta
  http-equiv="Content-Security-Policy"
  content="default-src 'self' https://www.googletagmanager.com https://www.google-analytics.com ;
           script-src 'self' https://www.googletagmanager.com https://www.google-analytics.com 'unsafe-eval' 'unsafe-inline';
           img-src 'self' https://www.google-analytics.com https://www.googletagmanager.com data: blob:;">
</head>

```

## Conclusion

Google Analytics is a great tool for tracking user activity and we have seen how it integrates into an Angular application to track page views and events. We have also seen a small example of how events tracking could be used and I hope that this has given you a few ideas on how you could integrate it into your own projects and use that data to optimize, market and evolve.
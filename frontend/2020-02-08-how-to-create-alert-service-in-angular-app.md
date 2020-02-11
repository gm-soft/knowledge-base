# Angular 8. Alert (Toaster) Notifications - https://jasonwatmore.com

Source: [https://jasonwatmore.com](https://jasonwatmore.com/post/2019/07/05/angular-8-alert-toaster-notifications)

## List of necessary files

To add alerts to your Angular 8 application you'll need to copy the /src/app/_alert folder and contents from the example project, the folder contains the alert module and associated files, including:

- `alert.component.html` - alert component template that contains the html for displaying alerts.
- `alert.component.ts` - alert component with the logic for displaying alerts.
- `alert.model.ts` - alert model class that defines the properties of an alert, it also includes the AlertType enum that defines the different types of alerts.
- `alert.module.ts` - alert module that encapsulates the alert component so it can be imported by the app module.
- `alert.service.ts` - alert service that can be used by any angular component or service to send alerts to alert components.
- `index.ts` - barrel file that re-exports the alert module, service and model so they can be imported using only the folder path instead of the full path to each file, and also enables importing from multiple files with a single import.

## Import the Alert Module into your App Module

To make the alert component available to your Angular 8 application you need to add the AlertModule to the imports array of your App Module (app.module.ts). See the app module from the example app below, the alert module is imported on line 4 and added to the imports array of the app module on line 14.

```typescript

import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';

import { AlertModule } from './_alert';
import { appRoutingModule } from './app.routing';

import { AppComponent } from './app.component';
import { HomeComponent } from './home';
import { MultiAlertsComponent } from './multi-alerts';

@NgModule({
    imports: [
        BrowserModule,
        AlertModule,
        appRoutingModule
    ],
    declarations: [
        AppComponent,
        HomeComponent,
        MultiAlertsComponent
    ],
    bootstrap: [AppComponent]
})
export class AppModule { }

```

## Add the <alert></alert> tag where you want alerts to be displayed

Add the alert component tag wherever you want alert messages to be displayed. The alert component accepts an optional id attribute if you want to display multiple alerts in different locations (see the Multiple Alerts page in the example above). An alert component instance with an id will display any messages sent to the alert service with the same alertId specified, e.g. alertService.error('something broke!', 'alert-1'); will send an error message to the alert component with id="alert-1".

The app component template in the example /src/app/app.component.html contains a global alert tag without an id above the router-outlet tag, this alert instance displays any messages sent to the alert service without an alertId specified, e.g. alertService.success('you won!'); will send a success message to the global alert without an id.

```html

<!-- main app container -->
<div class="jumbotron">
    <div class="container text-center">
        <alert></alert>
        <router-outlet></router-outlet>
    </div>
</div>

```

## Displaying Alert / Toaster Notifications in Your Angular 8 App

Once you've added support for alerts / toaster notifications to your app by following the previous steps, you can trigger alert notifications from any component in your application by simply injecting the alert service and calling one of it's methods for displaying different types of alerts: success(), error(), info() and warn().

Here is the home component from the example app that contains methods that pass example notification messages to the alert service when each of the buttons is clicked. In a real world application alert notifications can be triggered by any type of event, for example an error from an http request or a success message after a user profile is saved.

```typescript

import { Component } from '@angular/core';

import { AlertService } from '../_alert';

@Component({ templateUrl: 'home.component.html' })
export class HomeComponent {
    constructor(private alertService: AlertService) { }

    success(message: string) {
        this.alertService.success(message);
    }

    error(message: string) {
        this.alertService.error(message);
    }

    info(message: string) {
        this.alertService.info(message);
    }

    warn(message: string) {
        this.alertService.warn(message);
    }

    clear() {
        this.alertService.clear();
    }
}

```

And here is the home component template containing the buttons that are bound to the above methods.

```html

<h1>Angular 8 Alerts</h1>
<button class="btn btn-success m-1" (click)="success('Success!!')">Success</button>
<button class="btn btn-danger m-1" (click)="error('Error :(')">Error</button>
<button class="btn btn-info m-1" (click)="info('Some info....')">Info</button>
<button class="btn btn-warning m-1" (click)="warn('Warning: ...')">Warn</button>
<button class="btn btn-default m-1" (click)="clear()">Clear</button>

```

## Breakdown of the Angular 8 Alert / Toaster Notification Code

Below is a breakdown of the pieces of code used to implement the alert / toaster notification example in Angular 8, you don't need to know the details of how it all works to use the alerts in your project, it's only if you're interested in the nuts and bolts or if you want to modify the code or behaviour.

### Alert Service

The alert service (/src/app/_alert/alert.service.ts) acts as the bridge between any component in an Angular application and the alert component that actually displays the alert / toaster messages. It contains methods for sending and clearing alert messages, it also subscribes to the router NavigationStart event to automatically clear alert messages on route change, unless the keepAfterRouteChange flag is set to true, in which case the alert messages survive a single route change and are cleared on the next route change.

The service uses the RxJS Observable and Subject classes to enable communication with other components, for more information on how this works see [Angular 8 - Communicating Between Components with Observable & Subject](https://jasonwatmore.com/post/2019/06/21/angular-8-communicating-between-components-with-observable-subject).

```typescript

import { Injectable } from '@angular/core';
import { Router, NavigationStart } from '@angular/router';
import { Observable, Subject } from 'rxjs';
import { filter } from 'rxjs/operators';

import { Alert, AlertType } from './alert.model';

@Injectable({ providedIn: 'root' })
export class AlertService {
    private subject = new Subject<Alert>();
    private keepAfterRouteChange = false;

    constructor(private router: Router) {
        // clear alert messages on route change unless 'keepAfterRouteChange' flag is true
        this.router.events.subscribe(event => {
            if (event instanceof NavigationStart) {
                if (this.keepAfterRouteChange) {
                    // only keep for a single route change
                    this.keepAfterRouteChange = false;
                } else {
                    // clear alert messages
                    this.clear();
                }
            }
        });
    }

    // enable subscribing to alerts observable
    onAlert(alertId?: string): Observable<Alert> {
        return this.subject.asObservable().pipe(filter(x => x && x.alertId === alertId));
    }

    // convenience methods
    success(message: string, alertId?: string) {
        this.alert(new Alert({ message, type: AlertType.Success, alertId }));
    }

    error(message: string, alertId?: string) {
        this.alert(new Alert({ message, type: AlertType.Error, alertId }));
    }

    info(message: string, alertId?: string) {
        this.alert(new Alert({ message, type: AlertType.Info, alertId }));
    }

    warn(message: string, alertId?: string) {
        this.alert(new Alert({ message, type: AlertType.Warning, alertId }));
    }

    // main alert method    
    alert(alert: Alert) {
        this.keepAfterRouteChange = alert.keepAfterRouteChange;
        this.subject.next(alert);
    }

    // clear alerts
    clear(alertId?: string) {
        this.subject.next(new Alert({ alertId }));
    }
}

```

### Alert Component

The alert component (/src/app/_alert/alert.component.ts) controls the adding & removing of alerts in the UI, it maintains an array of alerts that are rendered by the component template.

The ngOnInit method subscribes to the observable returned from the alertService.onAlert() method, this enables the alert component to be notified whenever an alert message is sent to the alert service and add it to the alerts array for display. Sending an alert with an empty message to the alert service tells the alert component to clear the alerts array.

The ngOnDestroy() method unsubscribes from the alert service when the component is destroyed to prevent memory leaks from orphaned subscriptions.

The removeAlert() method removes the specified alert object from the array, it allows individual alerts to be closed in the UI.

The cssClass() method returns a corresponding bootstrap alert class for each of the alert types, if you're using something other than bootstrap you can change the CSS classes returned to suit your application.

```typescript

import { Component, OnInit, OnDestroy, Input } from '@angular/core';
import { Subscription } from 'rxjs';

import { Alert, AlertType } from './alert.model';
import { AlertService } from './alert.service';

@Component({ selector: 'alert', templateUrl: 'alert.component.html' })
export class AlertComponent implements OnInit, OnDestroy {
    @Input() id: string;

    alerts: Alert[] = [];
    subscription: Subscription;

    constructor(private alertService: AlertService) { }

    ngOnInit() {
        this.subscription = this.alertService.onAlert(this.id)
            .subscribe(alert => {
                if (!alert.message) {
                    // clear alerts when an empty alert is received
                    this.alerts = [];
                    return;
                }

                // add alert to array
                this.alerts.push(alert);
            });
    }

    ngOnDestroy() {
        // unsubscribe to avoid memory leaks
        this.subscription.unsubscribe();
    }

    removeAlert(alert: Alert) {
        // remove specified alert from array
        this.alerts = this.alerts.filter(x => x !== alert);
    }

    cssClass(alert: Alert) {
        if (!alert) {
            return;
        }

        // return css class based on alert type
        switch (alert.type) {
            case AlertType.Success:
                return 'alert alert-success';
            case AlertType.Error:
                return 'alert alert-danger';
            case AlertType.Info:
                return 'alert alert-info';
            case AlertType.Warning:
                return 'alert alert-warning';
        }
    }

```

### Alert Component Template

The alert component template (/src/app/_alert/alert.component.html) renders an alert message for each alert in the alerts array using the Angular *ngFor directive.

Bootstrap 4 is used for styling the alerts / toaster notifications in the example, you can change the HTML and CSS classes in this template to suit your application if you're not using Bootstrap.

```html

<div *ngFor="let alert of alerts" class="{{cssClass(alert)}} alert-dismissable">
    {{alert.message}}
    <a class="close" (click)="removeAlert(alert)">&times;</a>
</div>

```

### Alert Model and Alert Type Enum

The Alert model (/src/app/_alert/alert.model.ts) defines the properties of each alert object, and the AlertType enum defines the types of alerts allowed in the application.

```typescript

export class Alert {
    type: AlertType;
    message: string;
    alertId: string;
    keepAfterRouteChange: boolean;

    constructor(init?:Partial<Alert>) {
        Object.assign(this, init);
    }
}

export enum AlertType {
    Success,
    Error,
    Info,
    Warning
}

```

### Alert Module

The AlertModule (/src/app/_alert/alert.module.ts) encapsulates the alert component so it can be imported and used by other Angular modules.

```typescript

import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

import { AlertComponent } from './alert.component';

@NgModule({
    imports: [CommonModule],
    declarations: [AlertComponent],
    exports: [AlertComponent]
})
export class AlertModule { }

```

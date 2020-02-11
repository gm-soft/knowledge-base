# SPA Authentication using OpenID Connect, Angular CLI and oidc-client

Source: [https://www.scottbrady91.com/](https://www.scottbrady91.com/Angular/SPA-Authentiction-using-OpenID-Connect-Angular-CLI-and-oidc-client)

---

OpenID Connect is the go to protocol for modern authentication, especially when using Single Page Applications, or client-side applications in general. A library I often recommend to clients is oidc-client, a plain JavaScript library that is part of the IdentityModel OSS project. This handles all of the necessary protocol interactions with an OpenID Connect Provider, including token validation (which strangely some libraries neglect), and is a [certified OpenID Connect Relying Party](https://openid.net/certification/#RPs) conforming to the implicit RP and config RP profiles.

In this article, we are going to walk through a basic authentication scenario using the Angular CLI and the oidc-client library, during which we will authenticate a user, and then use an access token to access an OAuth protected API. This will use the implicit flow, where all tokens pass via the browser (something to always remember when dealing with code executing on the client, because the application cannot be trusted with features such as long lived tokens, refresh tokens or client secrets).

> Recommendations on which flow to use has changed ever so slightly. I recommend sticking with this article for now, and then giving the amendment a read: “[Migrating oidc-client-js to use the OpenID Connect Authorization Code Flow and PKCE](https://www.scottbrady91.com/Angular/Migrating-oidc-client-js-to-use-the-OpenID-Connect-Authorization-Code-Flow-and-PKCE)”. The migration path is trivial.

## Angular CLI Initialization

o keep this tutorial simple, we’re going to use the Angular CLI to create our Angular application along with basic routing. If you’re not using the Angular CLI, that’s fine, the OpenID Connect implementation specifics of this article applies to all Angular 4 applications.

So, if you haven’t already, install the Angular CLI as a global package:

`npm install -g @angular/cli`

We can then create a new application with routing already set up, for now skipping tests:

`ng new angular4-oidcclientjs-example –routing -skip-tests`

This will initialise everything we need to get started with our app and continue with this tutorial. You should already be able to run the application by navigating to the project (cd angular4-oidcclientjs-example) and running:

`ng serve`

And now if we navigate to our site (which by default runs on http://localhost:4200), we should see a splash screen with something like “Welcome to the app!”.

## Protected Component & Route Guard

### Protected Component

So, let’s start by creating a new page/component that requires a user to be authenticated in order to access it. We can generate the component using the Angular CLI command:

`ng generate component protected`

This will automatically add the component to our app.module, however we will need to manually add this component to our routing so that we can access it. To do this we need to make our app-routing.module look something like this:

```typescript

import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

import { ProtectedComponent } from './protected/protected.component';

const routes: Routes = [
    {
        path: '',
        children: []
    },
    {
        path: 'protected',
        component: ProtectedComponent
    }
];

@NgModule({
    imports: [RouterModule.forRoot(routes)],
    exports: [RouterModule]
})
export class AppRoutingModule { }

```

Here we have imported the component, and then registered a route for it with a url path of /protected.

Now let’s update our app.component.html to look like the following so that we can start testing our application:

```html

<h3><a routerLink="/">Home</a> | <a routerLink="/protected">Protected</a></h3>
<h1>
  {{title}}
</h1>
<router-outlet></router-outlet>

```

### Route Guard
Now that we have a page to protect, let’s do exactly that and protect it! We can do this using a route guard, with a guard type of CanActivate. This means that the guard will be able to decide whether or not a route can be activated based on some logic that we define. We’ll implement this fully in a bit, but for now we’ll leave it hardcoded to return false, which will prevent access to our protected route.

We can create a new route guard using the Angular CLI with:

`ng generate service services\authGuard`

We then need to import CanActivate from angular/router, make our service implement it, and then have the method return false. At a minimum, your route guard should look like the below, however you are welcome to implement the full interface.

```typescript

import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';

@Injectable()
export class AuthGuardService implements CanActivate {
        canActivate(): boolean {
            return false;
        }
}
Now we need to register our route guard in the Provider section of app.module’s NgModule, as this is not done for us automatically:

import { AuthGuardService } from './services/auth-guard.service';

@NgModule({
    // declarations, imports, etc.
    providers: [AuthGuardService]
})

```

We can then register the guard with our protected route in app-routing.module like so:

```typescript

import { AuthGuardService } from './services/auth-guard.service';

const routes: Routes = [
  // other routes
  {
    path: 'protected',
    component: ProtectedComponent,
    canActivate: [AuthGuardService]
  }
];

```

And now if we rerun the app and try to access the protected page, we should no longer be successful.

## Authentication using oidc-client

Now that we have our resource to protect and our guard, let’s create a service that can handle authentication and manage user sessions. To do this let’s first create a new service called AuthService:

`ng generate service services\auth`

And again register it as a provider within app.module:

```typescript

import { AuthService } from './services/auth.service';

@NgModule({
    // declarations, imports, etc.
    providers: [AuthGuardService, AuthService]
})

```

To handle all interactions with our OpenID Connect Provider, let’s bring in oidc-client. We can pull this in as a dependency in our package.json file with:

`"oidc-client": "^1.3.0"`

And we’ll also need its peer dependency of:

`"babel-polyfill": "^6.23.0"`

Don’t forget to make sure they install before continuing (npm update).

We now need to import UserManager, UserManagerSettings, and User into our auth service from the oidc-client library, like so:

```typescript

import { UserManager, UserManagerSettings, User } from 'oidc-client';

```

### UserManager

Our entry point into the oidc-client library is the UserManager. This is where all of our interactions with the oidc-client library will take place. Another option to use is OidcClient, but this only manages protocol support. For this article, we want the full user management provided by the UserManager.

The UserManager’s constructor requires a settings object of UserManagerSettings. For now we are going to hardcode these settings, but in production they should be initialized using your environment configuration.

```typescript

export function getClientSettings(): UserManagerSettings {
    return {
        authority: 'http://localhost:5555/',
        client_id: 'angular_spa',
        redirect_uri: 'http://localhost:4200/auth-callback',
        post_logout_redirect_uri: 'http://localhost:4200/',
        response_type:"id_token token",
        scope:"openid profile api1",
        filterProtocolClaims: true,
        loadUserInfo: true
    };
}

```

These settings should be recognisable to you if you have past experience with OpenID Connect Providers, but to clarify:

- **authority** is the URL of our OpenID Connect Provider
- **client_id** is the client application’s identifier registered within the OpenID Connect Provider
- **redirect_uri** is the client’s registered URI where all tokens will be sent to from the OpenID Connect Provider
- **response_type** can be thought of as the token types requested, which in this case is an identity token that represents the authenticated user and an access token to give us access to our protected resources. The other option here is code which is unsuitable for client side/in-browser applications, as it requires client credentials to be swapped for tokens
- **scope** is the scoped access which our application requires. In this case, we are asking for two identity scopes: openid and profile, which will allow us access to certain claims about the user, and one API scope: api1, which will allow us access to an API protected by this OpenID Connect Provider

These settings are required to create a UserManager. We’ve also included a few optional settings of:

- **post_logout_redirect_uri** which is a registered URI that the OpenID Connect provider can redirect a user to once they log out
- **filterProtocolClaims** which prevents protocol level claims such as nbf, iss, at_hash, and nonce from being extracted from the identity token as profile data. These claims aren’t typically of much use outside of token validation
- **loadUserInfo** allows the library to automatically call the OpenID Connect Provider’s User Info endpoint using the received access token, in order to access additional identity data about the authenticated user. This is true by default.

Currently we are using the OpenID Connect metadata endpoint for automatic discovery, but if this is not an option for you (maybe the discovery endpoint does not support CORS) the UserManager can be manually configured. Check out the [configuration section](https://github.com/IdentityModel/oidc-client-js/wiki#configuration) of the oidc-client documentation.

By default, the oidc-client will use the browsers session storage. This can be changed to local storage, however this can have privacy implications in some countries, as you would be storing personal information to disk. To switch to using local storage, you’ll need to import WebStorageStateStore and set the userStore property UserManagerSettings to:

```typescript

userStore: new WebStorageStateStore({ store: window.localStorage })

```

I used [IdentityServer](https://www.scottbrady91.com/Identity-Server/Getting-Started-with-IdentityServer-4) 4 as my OpenID Connect Provider when creating this tutorial, with the following configuration:

```csharp

new Client {
    ClientId = "angular_spa",
    ClientName = "Angular 4 Client",
    AllowedGrantTypes = GrantTypes.Implicit,
    AllowedScopes = new List<string> { "openid", "profile", "api1" },
    RedirectUris = new List<string> { "http://localhost:4200/auth-callback" },
    PostLogoutRedirectUris = new List<string> { "http://localhost:4200/" },
    AllowedCorsOrigins = new List<string> { "http://localhost:4200" },
    AllowAccessTokensViaBrowser = true
}

```

Within our auth service, initialize a new UserManager using your configured settings:

```typescript

private manager = new UserManager(getClientSettings());

```

Next, we’re going to create another local variable for the current user which we can initialize in the services constructor:

```typescript

private user: User = null;

constructor() {
    this.manager.getUser().then(user => {
        this.user = user;
    });
}

```

Here we are using the oidc-client getUser method. This loads in the current authenticated user, by looking in the configured store (in this case session storage). This returns a promise to load the user, so we’ve just saved this locally so that it’s easier to get to later. We’ll see this User object in action throughout this service.

### AuthService

Now we’re going to create five methods: isLoggedIn, getClaims, getAuthorizationHeaderValue, startAuthentication & completeAuthentication.

We’ll start with isLoggedIn. Here we will check if we have a user and if we do, well check if it is still valid. This can be done by using the expired property, which will calculate if the user’s access token for the user has expired or not.

```typescript

isLoggedIn(): boolean {
    return this.user != null && !this.user.expired;
}

```

getClaims is going to simply return the claims attached to the user, available on the profile property on the User object. Since we have set filterProtocolClaims to true, this will mostly be claims that make sense to your average user.

```typescript

getClaims(): any {
    return this.user.profile;
}

```

getAuthorizationHeaderValue is going to generate an authorization header from the User object. This requires the type of token (its scheme, probably Bearer) and the access token itself. We’ll see this in action later when we call an API, but for now it will look like:

```typescript

getAuthorizationHeaderValue(): string {
    return `${this.user.token_type} ${this.user.access_token}`;
}

```

To do the heavy lifting for protocol interaction, we now need startAuthentication and completeAuthentication. These will handle the OpenID Connect authentication requests for us, using the oidc-client signinRedirect and signinRedirectCallback methods which, when called upon, will automatically redirect users to our OpenID Connect provider using requests configured by our UserManagerSettings. An alternative to this would be to use signinPopup and signinPopupCallback which will open a new window for the request instead of a redirection.

```typescript

startAuthentication(): Promise<void> {
    return this.manager.signinRedirect();
}

completeAuthentication(): Promise<void> {
    return this.manager.signinRedirectCallback().then(user => {
        this.user = user;
    });
}

```

The signInRedirect method will generate the authorization request to our OpenID Connect Provider, handling the state and nonce, and, if required, call the metadata endpoint.

The callback method will receive and handle incoming tokens, including token validation. If loadUserInfo is set to true, it will also call the user info endpoint to get any extra identity data it has been authorized to access. This method returns a promise of the authenticated user, which we can then assign locally.

### Route Guard
Now let’s update our auth guard to use our newly created service. We’ll first check if the user is logged in, otherwise start authentication.

```typescript

import { Injectable } from '@angular/core';
import { CanActivate } from '@angular/router';

import { AuthService } from '../services/auth.service'

@Injectable()
export class AuthGuardService implements CanActivate {

    constructor(private authService: AuthService) { }

    canActivate(): boolean {
        if(this.authService.isLoggedIn()) {
            return true;
        }

        this.authService.startAuthentication();
        return false;
    }
}

```

## Callback Endpoint

And now we need one more component to complete authentication. This will be our auth callback component, giving us a way of retrieving the identity and access tokens returned from the OpenID Connect Provider and completing the authentication process using the oidc-client library. This is done by creating another component, which we’ll call auth-callback, and we'll use this as our redirect uri. So again, let’s use the Angular CLI:

`ng generate component auth-callback`

Here we need to import our auth service, pass it in via the constructor and in the ngOnInit call the auth service’s completeAuthentication method:

```typescript

constructor(private authService: AuthService) { }

ngOnInit() {
    this.authService.completeAuthentication();
}
And again, add the component to our routes. This path must be the a registered redirect uri within your OpenID Connect Provider.

import { AuthCallbackComponent } from './auth-callback/auth-callback.component';

const routes: Routes = [
    // other routes
    {
        path: 'auth-callback',
        component: AuthCallbackComponent
    }
];

```

Now when we try and access our protected area, we should be automatically redirected to our OpenID Connect provider. Once we authenticate, we’ll end up back in our application on our auth-callback page, with our tokens in the url fragment. If you check you session storage, you should see a new entry with a key of: `oidc.user:http://localhost:5555/:angular_spa` with a JSON value containing our identity token, access token, token type, and profile data.

That was a bit of a slog, but imagine what it would have been like if we weren’t using the Angular CLI.

### Redirects

Currently the user is being returned to the our callback url, which isn’t a great user experience. What could be done instead, is recording what page the user was trying to access before they were carted off to the OpenID Connect Provider, and then once they return to the application, the auth-callback component can redirect them back to that page from there. It’s up to you how you handle this, in the past I’ve seen people record the redirect path in session/local storage.

## Calling a Protected API

So far, we’ve protected an area within our application, forcing a user to authenticate before being authorized to access it, but what about calling an API that is protected by our OpenID Connect Provider (acting as an authorization server)? We are already requesting an access token as part of authentication, so let’s use this to authorize a request to an API.

First, let’s create a new component that will call our API:

`ng generate component call-api`

and add it with the route of /call-api, protected by our auth guard:

```typescript

import { CallApiComponent } from './call-api/call-api.component';

const routes: Routes = [
    // other routes
    {
        path: 'call-api',
        component: CallApiComponent,
        canActivate: [AuthGuardService]
    }
];

```

We also need to import HttpClientModule, done in app.module:

```typescript

import { HttpClientModule } from '@angular/common/http';

@NgModule({
    // declarations, providers, etc.
    imports: [HttpClientModule]
})

```

Inside this component we then need to import our security service and pass it in through the constructor, along with some imports from angular/common/http so that we can make HTTP requests:

```typescript

import { Component, OnInit } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';

import { AuthService } from '../services/auth.service'

@Component({
    selector: 'app-call-api',
    templateUrl: './call-api.component.html',
    styleUrls: ['./call-api.component.css']
})
export class CallApiComponent implements OnInit {

    constructor(private http: Http, private authService: AuthService) { }
    ngOnInit() {
    }
}

```

Now in our ngOnInit we are going to setup our authorization header and then call our API. We’ll take the response and set it as a local property.

```typescript

export class CallApiComponent implements OnInit {
    response: Object;
    constructor(private http: HttpClient, private authService: AuthService) { }

    ngOnInit() {
        let headers = new HttpHeaders({ 'Authorization': this.authService.getAuthorizationHeaderValue() });

        this.http.get("http://localhost:5555/api", { headers: headers })
          .subscribe(response => this.response = response);
    }
}

```

The API we are calling here simply returns some text and requires a bearer token issued by `http://localhost:5555`, with a scope and audience of api1.

Now in the components html we’re just going display that response:

```html

<p>
    Response: {{response}}
</p>

```

And if we update our homepage to include a link to this functionality:

```html

<h3>
    <a routerLink="/">Home</a>
    | <a routerLink="/protected">Protected</a> 
    | <a routerLink="/call-api">Call API</a>
</h3>
<h1>
  {{title}}
</h1>
<router-outlet></router-outlet>

```

## Token Expiration

Currently, if your access token expires one of two things will happen: the auth service will detect you as logged when you next try and Access a protected page, or you will receive a 401 unauthorised from your API.

The first scenario is fine, as our auth service will automatically redirect us to our identity provider for authentication and return us fresh tokens as a result. However the second scenario could lead to data loss if we had, for example, just filled out a form. Since we can't use refresh token when using the implicit flow, we have to take a different approach. This is where the silent refresh feature of the OIDC-client comes into play, which you can read about in my “[Silent Refresh - Refreshing Access Tokens when using the Implicit Flow](https://www.scottbrady91.com/OpenID-Connect/Silent-Refresh-Refreshing-Access-Tokens-when-using-the-Implicit-Flow)” article.

## Cordova, Ionic, & Electron

If you are turning your JavaScript application into a native/mobile application (e.g. using Cordova, Ionic, or Electron) then **do not use the implicit flow**. This flow is only ever suitable for browser-based applications. Using the implicit flow for native applications is unsecure, no matter what other articles may tell you. Instead, use the hybrid or authorization code flows along with PKCE, following best practices from [RFC 8252](https://tools.ietf.org/html/rfc8252).

## Source Code

You can find the full source code for [the Angular application and a supporting instance of IdentityServer 4 and API](https://github.com/scottbrady91/Angular4-OidcClientJs-Example/tree/implicit) on GitHub.

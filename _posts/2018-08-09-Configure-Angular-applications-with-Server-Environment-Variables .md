---
layout: post
title: Configure Angular apllication with Server Environment Variables.
excerpt: "To configure an angular client to receive environment variables as set on the host Server. This allows us to promote our code base between environments without altering our client files. This could also be used in a scenario where we take advantage of Azure slotting to provide slot persistent settings to the client during a swap."
tags: [Angular] [Node JS]
---

# Configure your Angular applications with Server Environment Variables

To configure an angular client to receive environment variables as set on the host Server. This allows us to promote our code base between environments without altering our client files. This could also be used in a scenario where we take advantage of Azure slotting to provide slot persistent settings to the client during a swap.

## Application Settings setup

In the application settings page in the Azure portal we first define the settings that we want to provide to the end client application. In this example we create an application environment setting named rest_url and give it a specific value.

## Angular client setup

Create a class definition called AppSettings that will retain the values from the setting keys that we specified in Azure. This type will later be injected to dependent modules in our client.

```js
import { Injectable } from '@angular/core';

@Injectable()
export class AppSettings {
  rest_url: string;
}
```

Now, when we start the client code in the browser the first thing that needs to happen is a blocking call must be made to the configuration endpoint. This is necessary to ensure that all subsequent code can receive the values of the application settings.

The app config service is used to retrieve the json object from the host server. Values received from the service are copied to the instance of the appSettings object that will be injected into our dependencies. The httpClient call uses a promise rather than an observable in order to block until the method has completed.

```js
import {Injectable, isDevMode} from '@angular/core';
import {HttpClient} from '@angular/common/http';
import {AppSettings} from '../app.settings';
import {environment} from '../../../environments/environment';

@Injectable()
export class AppConfigService {
  constructor(private http: HttpClient, private appSettings: AppSettings) { }

  load(): Promise<void> {
    const configUrl = environment.configEndpoint;
    console.log('getting configuration values from ' + configUrl);

    return this.http.get(configUrl)
      .toPromise()
      .then(data => {
        for (const key in data) {
          if (this.appSettings[key] !== 'undefined') {
            this.appSettings[key] = data[key];
          }
        }
        console.log('AppConfigService loaded() ' + JSON.stringify(data));
      });
  }
}
```

Create an angular module called AppConfigLoadModule. This will call our AppConfigService load() method as part of the boot strapping process.

```js
import { NgModule, APP_INITIALIZER } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
import { AppConfigService } from '../services/app.config.service';

const appInitializerFn = (appConfig: AppConfigService) => {
  return () => appConfig.load();
};

@NgModule({
  imports: [HttpClientModule],
  providers: [
    AppConfigService,
    { provide: APP_INITIALIZER, useFactory: appInitializerFn, deps: [AppConfigService], multi: true }
  ]
})
export class AppConfigLoadModule { }
```

The angular app module must load the configuration module, to do this inject the AppConfigModule as an import within App.Module
Register AppSettings as a provider to make it available anywhere within the application.

```js
import {NgModule} from '@angular/core';
import {BrowserModule} from '@angular/platform-browser';
import {AppComponent} from './app.component';
import {AppConfigLoadModule} from './shared/modules/app.config.load.module';
import {AppSettings} from './shared/app.settings';

@NgModule({
  imports: [ BrowserModule, AppConfigLoadModule ],
  declarations: [ AppComponent ],
  bootstrap: [ AppComponent ],
  providers: [ AppSettings ]
})
export class AppModule { }
```

## Server.js /config endpoint

Add the following method to server.js, This provides a server side endpoint that we can call from our client code to gather the server environment settings. We are surfacing specific application settings from the process Environment object and encoding these as json for our client to consume. There is an implicit convention that our client application will make a call to https://[host]/config on the server that hosts the application files.

```js
  app.get('/config', function (req, res, next) {
    res.setHeader('Content-Type', 'application/json');
    res.send(JSON.stringify(
      { rest_url: process.env.rest_url }
    ));
  });
```

## To use the appSettings type

We inject the AppSettings type into our dependent objects. This types properties are set by the AppConfigService earlier and can now be used to direct the behavior of our component.

```js
import {Component} from '@angular/core';
import {AppSettings} from './shared/app.settings';

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html'
})
export class AppComponent {
  constructor(appSettings: AppSettings) {
    //use the appSettings values here
  }
}
```

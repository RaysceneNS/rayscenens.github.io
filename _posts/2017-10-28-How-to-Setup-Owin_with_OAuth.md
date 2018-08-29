---
layout: post
title: How-to Setup OpenIDConnect with Owin MVC
excerpt: "Open ID Connect, is a protocol that extends Open Auth with basic profile support. Implementing it in an Owin MVC project is straight forward."
tags: [OAuth, Owin]
---

# OpenIDConnect in Owin MVC project

## Overview

Using Azure Active Directory is a great way to offload the task of validating users for your web applications. The benefits to this approach are many.

- We can implement single sign-on
- Advanced scenarios such as two factor are a configuration change away
- Our application code does not have to store users passwords (you did hash those properly right?)

## Register your application with Azure AD

 In order to setup your Owin Mvc applications to use the authorization services in AAD. You will first need to register the application in the Azure Active Directory. Here we will
 1. Login to the azure portal
 2. Select Azure Active Directory from the navigation, and click create 'App registration'
 3. Enter the name of our application, select Web app/api for the application type and most importantly enter the sign-on url. This is the base url of your web application e.g. ```https://my.net```
 4. Make note of the Application ID assigned from the wizard, we'll need this later on.


## Install the authorization handler

Next we will install the OpenIdConnect middleware in our Owin pipeline. This package will take care of requests that will be made to our app as a regular part of implementing the Auth flow.

![Authentication Flow](/assets/images/2017/10/28/Diagram1.png)

Install the following nuget packages into the project

```PowerShell
Microsoft.Owin.Security.OpenIdConnect
Microsoft.Owin.Security.OAuth
Microsoft.Owin.Security.Jwt
Microsoft.Owin.Security.Cookies
Microsoft.Owin.Host.SystemWeb
```

Insert a call to a ConfigureAuth function in  startup.cs file add the following call to ConfigureAuth():

```c#
    public partial class Startup
    {
        public void Configuration(IAppBuilder app)
        {
            ConfigureAuth(app);
        }
    }
```

Insert in app_Start.cs

```c#
    public partial class Startup
    {
        public void ConfigureAuth(IAppBuilder app)
        {
            var issuer = ConfigurationManager.AppSettings["_App_Service_Auth_URL"];
            var audience = ConfigurationManager.AppSettings["App_Service_ClientID"];
            var secret = TextEncodings.Base64.Encode(TextEncodings.Base64Url.Decode(ConfigurationManager.AppSettings["App_Service_ClientSecret"]));

            app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);

            app.UseCookieAuthentication(new CookieAuthenticationOptions());

            // Enable the application to use a cookie to store information for the signed in user
            app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
            {
                ClientId = CLIENT_ID_HERE,
                Authority = "https://login.microsoftonline.com/" + TENANT_ID_HERE,
                RedirectUri = redirectUrl,
                UseTokenLifetime = true,
                CallbackPath = new PathString("/signin-oidc"),
                Scope = "openid email profile",
                ResponseType = "id_token",
                Notifications = new OpenIdConnectAuthenticationNotifications
                {
                    AuthenticationFailed = OnAuthenticationFailed,
                }
            });
        }

        private Task OnAuthenticationFailed(AuthenticationFailedNotification<OpenIdConnectMessage, OpenIdConnectAuthenticationOptions> arg)
        {
            context.HandleResponse();
            context.Response.Redirect("/?errormessage=" + context.Exception.Message);
            return Task.FromResult(0);
        }
    }
```
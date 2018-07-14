---
layout: post
title: How-to setup Basic Authentication filter in an Asp.Net Web API project
tags: [Web.Api, Security]
---

# How-to setup Basic Authentication filter in an Asp.Net Web API

There is no built in support for Basic Authentication when creating a Web.Api controller. However adding this support is fairly straight forward.

The first thing we need to do is create a class that implements the IAuthenticationFilter interface and add it to our project. This class will provide implementations of Task AuthenticateAsync(HttpAuthenticationContext context, CancellationToken cancellationToken); and Task ChallengeAsync(HttpAuthenticationChallengeContext context, CancellationToken cancellationToken);

The AuthenticateAsync method is invoked on every call to the web api controller that is associated to the filter. Within this call 
we will determine if the proper authentication header is supplied from the client and passed to us. If the auth header is valid then a generic principal is set on the HttpContext _this could be any type of principal required for your security setup_.

```c#
public Task AuthenticateAsync(HttpAuthenticationContext context, CancellationToken cancellationToken)
{
    var authorization = context.Request.Headers.Authorization;
    if (authorization != null &&
        string.Equals(authorization.Scheme, "Basic", StringComparison.OrdinalIgnoreCase) &&
        !string.IsNullOrEmpty(authorization.Parameter))
    {
        if (ExtractUserNameAndPassword(authorization.Parameter, out var userName, out var password) &&
            userName == {VALID_USERNAME} && password == {VALID_PASSWORD})
        {
            var identity = new GenericIdentity(userName, "Basic");
            context.Principal = new GenericPrincipal(identity, null);
        }
        else
        {
            context.ErrorResult = new AuthenticationFailureResult("Invalid username or password", context.Request);
        }
    }
    else
    {
        context.ErrorResult = new AuthenticationFailureResult("Missing auth", context.Request);
    }
    return Task.FromResult(0);
}
```

This method extracts the user and password from the authorization header supplied by the client browser in the form

```HTTP
POST /api/controller HTTP/1.1
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
```

The rfc specification says that the supplied format must be encoded in ISO-8859-1 format. To decode this parameter the header is sent through the following function.

```c#
private static bool ExtractUserNameAndPassword(string authorizationParameter, out string userName, out string password)
{
    userName = null;
    password = null;
    byte[] credentialBytes;
    try
    {
        credentialBytes = Convert.FromBase64String(authorizationParameter);
    }
    catch (FormatException)
    {
        return false;
    }

    var encoding = System.Text.Encoding.GetEncoding("ISO-8859-1");
    var decodedCredentials = encoding.GetString(credentialBytes);
    if (string.IsNullOrEmpty(decodedCredentials))
    {
        return false;
    }

    int colonIndex = decodedCredentials.IndexOf(':');
    if (colonIndex == -1)
    {
        return false;
    }

    userName = decodedCredentials.Substring(0, colonIndex);
    password = decodedCredentials.Substring(colonIndex + 1);
    return true;
}
```

To handle the case where the authentication fails and we need to challenge the browser for the correct authentication. We install an IHttpActionResult in the following method.

```c#
public Task ChallengeAsync(HttpAuthenticationChallengeContext context, CancellationToken cancellationToken)
{
    var host = context.Request.RequestUri.DnsSafeHost;
    context.Result = new AddChallengeOnUnauthorizedResult(new AuthenticationHeaderValue("Basic", "realm=\"" + host + "\""), context.Result);
    return Task.CompletedTask;
}
```

The AddChallengeOnUnauthorizedResult will return a challenge to the browser if the status code is Unauthorized.

```c#
 public class AddChallengeOnUnauthorizedResult : IHttpActionResult
{
    private readonly AuthenticationHeaderValue _challenge;
    private readonly IHttpActionResult _innerResult;

    public AddChallengeOnUnauthorizedResult(AuthenticationHeaderValue challenge, IHttpActionResult innerResult)
    {
        _challenge = challenge;
        _innerResult = innerResult;
    }

    public async Task<HttpResponseMessage> ExecuteAsync(CancellationToken cancellationToken)
    {
        var response = await _innerResult.ExecuteAsync(cancellationToken);
        if (response.StatusCode == HttpStatusCode.Unauthorized && response.Headers.WwwAuthenticate.All(h => h.Scheme != _challenge.Scheme))
            response.Headers.WwwAuthenticate.Add(_challenge);
        return response;
    }
}
```

Finally to enable this filter for all web api requests we simply add the filter to the Filters collection. Within our webapiconfig.cs

```c#
    config.Filters.Add(new BasicAuthenticationFilter());
```
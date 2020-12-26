---
title: "Validate JWT Token signed with RS256 for use within Microsoft Dynamics 365 CRM"
tags: [Crypto,Dynamics]
---

How to secure an external web API call for use within Microsoft Dynamics 365 CRM Portals.

## Overview

With the April 2019 release of Microsoft Dynamics a new feature was added that provides an endpoint that can be used to obtain secure access tokens that contain user identity information. The addition of this feature provides a solution that can be used to call external Web API's on behalf of the user that is logged into the CRM application.

The authorization process uses OAuth implicit grant type flow.
![State Machine](/assets/images/2019/06/26/WebSequenceDiagram.svg)

## JWT Token Structure

The JWT consists of three parts separated by periods ( . )

- Header
- Payload
- Signature

### Header

The header is a base64 encoded JSON string. It typically encodes the signing algorithm used and the type of token.

example of the header:

``` json
{
  "typ": "JWT",
  "alg": "RS256"
}
```

### Payload

The payload is another base64 encoded JSON value. It contains the claims, which are statements about the current user. There are three types of claims: registered, public and private.

- Registered Claims: These are a set of predefined claims which are not mandatory but recommended, to provide a set of useful, interoperable claims. Some examples of these are:
  - iss (issuer)
  - exp (expiration time)
  - sub (subject)
  - aud (audience)
- Public Claims: These can be defined at will by those using JWTs. But to avoid collisions they should be defined in the IANA JSON web token registry.
- Private Claims: These are any claims that are agreed upon between parties and are neither registered or public.

An example payload:

``` json
{
  "sub": "99db51-xxx-xxx-155d03a715",
  "preferred_username": "customer",
  "phone_number": "(425) 555-5555",
  "given_name": "John",
  "family_name": "Doe",
  "email": "jdoe@contoso.com",
  "lp_sdes": "[{\"type\":\"ctmrinfo\",\"info\":{\"cstatus\":null,\"ctype\":\"contact\",\"customerId\":\"99db51a2715\",\"balance\":null,\"socialId\":null,\"imei\":\"(425) 555-5555\",\"userName\":\"customer\"}}]",
  "iat": 1556572581,
  "iss": "issuing_server.domain.com",
  "exp": 1556576182,
  "nbf": 1556572582
}
```

## Signature

Signed tokens can verify the integrity of the claims contained within it. Because the token has been signed with a public/private key pair, the signature also certifies that only the party holding the private key is the one that signed it.

Note that the token is not encrypted, all information within the token is presented in plain text. The claims within the token should not contain secret information and should only be sent over https to the external party.

The signature is a byte array that represents the RSA signing data that must be verified by the api to ensure that the token is authentic. In addition to verifying the signature the server should verify that the current time lies between the exp (expiry) and nbf (not before). The api must check that the iss (issuer) parameter matches the expected value.

## Web Client Code

The following JavaScript demonstrates how a client can obtain a JWT token, and then pass this token to an external service via the http Authorization header.

``` javascript
var _jwtToken;
$(document).ready(function() {
    $.ajax({
        url: "https://PORTAL_SERVER/_services/auth/token",
        type: "GET",
        data: {
            response_type : "token"
        },
        success: function(token) {
            _jwtToken = token;
            callApi();
        }
    });
});

var callApi = function() {
    $.ajax({
        url: "https://EXTERNAL_SERVER/api/me",
        type: "GET",
        headers: {'Authorization':'Bearer ' + _jwtToken},
        success: function(data) {
            console.log(data);
        }
    });
}
```

## Web Api Token Validation

The following snippet demonstrates how a .Net web api may be setup to perform validation of the JWT token as sent by the client browser. Validation is necessary to ensure that the information contained within the token comes from a legitimate server and thus we can trust the claims contained within the token as authentic.

The CRM portal provides an endpoint at <https://{PORTAL_SERVER}/_services/auth/publickey>

The response from this endpoint is the public key that can be used to verify the token. The public key endpoint returns an RSA public key in PEM (Privacy Enhanced Mail) format

for example:

```console
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqFAmeuvnR6GM/B04NAFB
658rVEmmOIFj2z41Z22kBz6dImkDx5rIzxcgmslxG/SlGUqW0oTji6B7RJIMdjhB
ucZNhl3NodpQ88RR9oQkf...qivpb/yis7nQM1rWKhZpg3sMzzUTgaAKvSrLg
begFHCVGfwcsk5gtz97S2tW6owlmM0UQWc+nId0xAH4QiVcoQjMAsEJvkV9CjY69
IQIDAQAB
-----END PUBLIC KEY-----
```

The token validation can be achieved by using the JwtSecurityTokenHandler class as per this example.

``` c#
internal class TokenValidationHandler : DelegatingHandler
{
    private static readonly string _issuer = ConfigurationManager.AppSettings["ida:Issuer"];
    private SecurityKey _signingKey;

    protected async override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
    {
        // Get the jwt bearer token from the authorization header
        string jwtToken = null;
        if (request.Headers.Authorization != null)
        {
            jwtToken = request.Headers.Authorization.Parameter;
        }

        if (jwtToken == null)
        {
            return this.BuildResponseErrorMessage(HttpStatusCode.Unauthorized);
        }

        try
        {
            ClaimsPrincipal claimsPrincipal =
                new JwtSecurityTokenHandler().ValidateToken(jwtToken,
                new TokenValidationParameters
                {
                    // aud is not present in JWT currently
                    ValidateAudience = false,

                    // ensure iss is valid
                    ValidIssuer = "PORTAL_SERVER.microsoftcrmportals.com",
                    ValidateIssuer = true,

                    // verify that the token is valid by using the public key from the portal
                    IssuerSigningKey = signingKey,
                    ValidateIssuerSigningKey = true,

                    ValidateLifetime = true,
                    RequireExpirationTime = true,
                    RequireSignedTokens = true,
                },
                out var validatedToken);
        }
        catch (SecurityTokenValidationException)
        {
            return new HttpResponseMessage(HttpStatusCode.Unauthorized);
        }
        catch (Exception)
        {
            return new HttpResponseMessage(HttpStatusCode.InternalServerError);
        }
    }

    private SecurityKey RetrieveSigningKey()
    {
        if (_signingKey == null)
        {
            using (var webClient = new WebClient())
            {
                string url = string.Format("https://{0}/_services/auth/publickey", ConfigurationManager.AppSettings["ida:Issuer"]);
                using (var webStream = webClient.OpenRead(url))
                {
                    using (var streamReader = new System.IO.StreamReader(webStream))
                    {
                        var pemReader = new Org.BouncyCastle.OpenSsl.PemReader(streamReader);
                        RsaKeyParameters publicKeyParam = (RsaKeyParameters)pemReader.ReadObject();
                        _signingKey = new RsaSecurityKey(new RSAParameters
                        {
                            Modulus = publicKeyParam.Modulus.ToByteArrayUnsigned(),
                            Exponent = publicKeyParam.Exponent.ToByteArrayUnsigned()
                        });
                    }
                }
            }
        }
        return _signingKey;
    }
```

note: This relies on a signing key parameter, which is the RSA public key as loaded from the public key endpoint.We rely on an external library [BouncyCastle.Crypto] to read the PEM string, hopefully in the future the .Net framework will provide built in support for parsing PEM formatted signing keys natively.

Register the token validation handler within the web api like so

```c#
public class WebApiApplication : System.Web.HttpApplication
{
    protected void Application_Start()
    {
        GlobalConfiguration.Configure(WebApiConfig.Register);
        GlobalConfiguration.Configuration.MessageHandlers.Add(new TokenValidationHandler());
    }
}
```

With this validation handler installed we can interrogate the claim set within a web api controller.

```c#
Claim subject = ClaimsPrincipal.Current.FindFirst(ClaimTypes.NameIdentifier);
Claim given = ClaimsPrincipal.Current.FindFirst(ClaimTypes.GivenName);
Claim family = ClaimsPrincipal.Current.FindFirst(ClaimTypes.Surname);
return new string[] { subject.Value, given.Value, family.Value };
```

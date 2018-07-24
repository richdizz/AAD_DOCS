# Calling v1-secured APIs

In this topic, we will outline how applications registered with the Azure AD v2.0 endpoint can call into APIs registered with the Azure AD v1.0 endpoint. This includes all applications registered through the Microsoft Azure portal.

## Azure AD v1.0 and Tenancy
Both the Azure AD v1.0 endpoint (https://login.microsoftonline.com/{tenant}/oauth2/) and Azure AD v2.0 endpoint (https://login.microsoftonline.com/{tenant}/oauth2/v2.0/) have a `{tenant}` value in the path of the request. When performing OAuth flows, `common` is often used for the tenant in multi-tenant applications. It is important to recognize "multi-tenant" means different things between Azure AD v1.0 and v2.0 endpoints. Applications registered against the Azure AD v1.0 endpoint are exclusive to organizations and do not support sign-in from consumer accounts. Here `common` represents "all organizations". The Azure AD v2.0 endpoint introduced "converged authentication" for APIs that support both organizations and consumers (ex: Microsoft Graph). With the Azure AD v2.0 endpoint, `common` represents "all Microsoft personal (MSA) and all organizations". The Azure AD v2.0 endpoint offers more grainular multi-tenant options such as `organizations` and `consumers`.

It is not valid using `common` for the tenant to request tokens for an Azure AD v1.0 resource with the Azure AD v2.0 endpoint. Doing this will result in an error similar to below.

```
AADSTS90124: Resource 'https://analysis.windows.net/powerbi/api' (Microsoft.Azure.AnalysisServices) is not supported over the /common or /consumers endpoints. Please use the /organizations or tenant-specific endpoint.
```

As the error suggests, the Azure AD v2.0 endpoint should use `organizations` or tenant-specific endpoints (ex: `8eaef023-2b34-4da1-9baa-8bc8c9d6a490` or `contoso.onmicrosoft.com`) to get tokens for an Azure AD v1.0 resource. 

## Permission Scopes
Outside of well known permission scopes used by the Microsoft Graph (ex: `Files.Read`), the Azure AD v2.0 permission scope syntax uses the pattern `{app_uri}/{scope_string}`. For example, the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` permission scope for `View all Datasets` would be `https://analysis.windows.net/powerbi/api/Dataset.Read.All`, which is comprised of the Power BI app URI `https://analysis.windows.net/powerbi/api/` and the permission scope string `Dataset.Read.All`.

> As Azure AD works to converge the developer experience between the Azure AD v1.0 and v2.0 endpoints, you might find it difficult to figure out the app URI and scope strings of applications. This is partially because scopes are not used in Azure AD v1.0 OAuth flows and thus not as commonly used until now. We have [documented a number popular resource scopes](popular-v1-permissions.md) and [instructions for discovering any v1 permission scope string](discovering-v1-permission-scopes.md).

## Walkthrough
Below are the steps you need to take to bring your Azure AD v2.0 endpoint to communication with a service designed only for commercial accounts.

The basic steps required to use the OAuth 2.0 authorization code grant flow to get an access token from a service registered as an Azure AD v1.0 application are:

1. Get authorization
2. Get a token
3. Use the refresh token to get a new access token

## 1. Get authorization
The first step to getting an access token for many OpenID Connect and OAuth 2.0 flows is to redirect the user to your application's Azure AD v2.0 `/authorize` endpoint. Azure AD will sign the user in and ensure their consent for the permissions your app requests. In the authorization code grant flow, after consent is obtained, Azure AD will return an authorization_code to your app that it can redeem at the Azure AD v2.0 `/token` endpoint for an access token.

### Authorization request 
The following shows an example request to the `/authorize` v2.0 endpoint for an Azure AD v1.0 resource. In this example, Power BI is the Azure AD v1.0 resource referenced. 

With the Azure AD v2.0 endpoint, permissions are requested using the `scope` parameter. In this example, the Power BI permissions requested are _Dataset.Read.All_ (View all Datasets), which will allow the app to read all Power BI datasets for the signed-in user. The _offline\_access_ permission is requested so that the app can get a refresh token, which it can use to get a new access token when the current one expires.

```
// Line breaks for legibility only
// {tenant} should be "organizations" or tenant-specific for v1.0 resource

https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize?
client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=code
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&response_mode=query
&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2FDataset.Read.All%20offline_access
&state=12345
```
<!-- http://localhost/myapp/ ; https://analysis.windows.net/powerbi/api/ -->

| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  The allowed values are `common` for both Microsoft accounts and work or school accounts, `organizations` for work or school accounts only, `consumers` for Microsoft accounts only, and tenant identifiers such as the tenant ID or domain name. For Azure AD v1.0 resources, the allowed values are `organizations` and tenant identifiers such as the tenant ID or domain name. For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| response_type |required |Must include `code` for the authorization code flow. |
| redirect_uri |recommended |The redirect_uri of your app, where authentication responses can be sent and received by your app.  It must exactly match one of the redirect_uris you registered in the app registration portal, except it must be URL encoded.  For native and mobile apps, you should use the default value of `https://login.microsoftonline.com/common/oauth2/nativeclient`. |
| scope |required | A space-separated list of permissions that you want the user to consent to. This can be well-known scopes such as those used by the Microsoft Graph (ex: Files.Read), a combination of app URI and scope string (ex: `https://analysis.windows.net/powerbi/api/Dataset.Read.All`), or the resource default if permissions are pre-configured for the app (ex: `https://analysis.windows.net/powerbi/api/.default`). This may also include OpenID scopes. |
| response_mode |recommended |Specifies the method that should be used to send the resulting token back to your app.  Can be `query` or `form_post`. |
| state |recommended |A value included in the request that will also be returned in the token response.  It can be a string of any content that you wish.  A randomly generated unique value is typically used for [preventing cross-site request forgery attacks](http://tools.ietf.org/html/rfc6749#section-10.12).  The state is also used to encode information about the user's state in the app before the authentication request occurred, such as the page or view they were on. |

### Consent experience

At this point, the user will be asked to enter their credentials to authenticate with Azure AD. The v1.0 endpoint will also ensure that the user has consented to the permissions indicated in your v2.0 application's registration. If the user does not consent to any of those permissions and if an administrator has not previously consented on behalf of all users in the organization, Azure AD will ask the user to consent to the required permissions.

> **Try** this once with an AAD account and once with another MSA account. After signing in, your browser should be redirected to `https://localhost/myapp/` with a `code` in the address bar if done with an AAD account. This is because it specifically points to the organizations endpoint, so it's limited to AAD accounts.
>
> <a href="https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize?client_id=6731de76-14a6-49ae-97bc-6eba6914391e&response_type=code&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F&response_mode=query&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2FDataset.Read.All&state=12345" 
target="_blank">https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize...</a>
<!-- TODO: 1) explain it's only powerbi or 2) replace with something like outlook -->

### Authorization response
If the user consents to the permissions your app requested, the response will contain the authorization code in the `code` parameter. Here is an example of a successful response to the request above. Because the `response_mode` parameter in the request was set to `query`, the response is returned in the query string of the redirect URL.

```
GET http://localhost/myapp/?
code=M0ab92efe-b6fd-df08-87dc-2c6500a7f84d
&state=12345
```
| Parameter | Description |
| --- | --- |
| code |The authorization_code that the app requested. The app can use the authorization code to request an access token for the target resource.  Authorization_codes are very short lived, typically they expire after about 10 minutes. |
| state |If a state parameter is included in the request, the same value should appear in the response. The app should verify that the state values in the request and response are identical. |

## 2. Get a token
Your app uses the authorization `code` received in the previous step to request an access token by sending a `POST` request to the `/token` endpoint.

### Token request
```
// Line breaks for legibility only
// {tenant} should be "organizations" or tenant-specific for v1.0 resource

POST /{tenant}/oauth2/v2.0/token HTTP/1.1
Host: https://login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2FDataset.Read.All%20offline_access
&code=OAAABAAAAiL9Kn2Z27UubvWFPbm0gLWQJVzCTE9UkP3pSx1aXxUjq3n8b2JRLk4OxVXr...
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&grant_type=authorization_code
&client_secret=JqQX2PNo9bpM0uEihUPzyrh    // NOTE: Only required for web apps
```

| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  For Azure AD v1.0 resources, the allowed values are `organizations` and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| grant_type |required |Must be `authorization_code` for the authorization code flow. |
| scope |required |A space-separated list of permissions that you want the user to consent to. This can be well-known scopes such as those used by the Microsoft Graph (ex: Files.Read), a combination of app URI and scope string (ex: `https://analysis.windows.net/powerbi/api/Dataset.Read.All`), or the resource default if permissions are pre-configured for the app (ex: `https://analysis.windows.net/powerbi/api/.default`). This may also include OpenID scopes. |
| code |required |The authorization_code that you acquired in the first leg of the flow. |
| redirect_uri |required |The same redirect_uri value that was used to acquire the authorization_code. |
| client_secret |required for web apps |The application secret that you created in the app registration portal for your app.  It should not be used in a native app, because client_secrets cannot be reliably stored on devices.  It is required for web apps and web APIs, which have the ability to store the client_secret securely on the server side. |

### Token response
Although the access token is opaque to your app, the response contains a list of the permissions that the access token is good for in the `scope` parameter. 

```
{
    "token_type": "Bearer",
    "scope": "https://analysis.windows.net/powerbi/api/Dataset.Read.All",
    "expires_in": 3600,
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...",
    "refresh_token": "AwABAAAAvPM1KaPlrEqdFSBzjqfTGAMxZGUTdM0t4B4..."
}
```

| Parameter | Description |
| --- | --- |
| token_type |Indicates the token type value. The only type that Azure AD supports is Bearer |
| scope |A space separated list of the permissions that the access_token is valid for. |
| expires_in |How long the access token is valid (in seconds). |
| access_token |The requested access token. Your app can use this token to call into the resource it was requested for. |
| refresh_token |An OAuth 2.0 refresh token. Your app can use this token to acquire additional access tokens after the current access token expires.  Refresh tokens are long-lived, and can be used to retain access to resources for extended periods of time.  For more detail, refer to the [v2.0 token reference](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-tokens). |

## 3. Use the refresh token to get a new access token

Access tokens are short lived, and you must refresh them after they expire to continue accessing resources.  You can do so by submitting another `POST` request to the `/token` endpoint, this time providing the `refresh_token` instead of the `code`.

### Request
```
// Line breaks for legibility only
// {tenant} should be "organizations" or tenant-specific for v1.0 resource

POST /{tenant}/oauth2/v2.0/token HTTP/1.1
Host: https://login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2FDataset.Read.All
&refresh_token=OAAABAAAAiL9Kn2Z27UubvWFPbm0gLWQJVzCTE9UkP3pSx1aXxUjq...
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&grant_type=refresh_token
&client_secret=JqQX2PNo9bpM0uEihUPzyrh      // NOTE: Only required for web apps
```

| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  For Azure AD v1.0 resources, the allowed values are `organizations` and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| grant_type |required |Must be `refresh_token`. |
| scope |required |A space-separated list of permissions that you want the user to consent to. This can be well-known scopes such as those used by the Microsoft Graph (ex: Files.Read), a combination of app URI and scope string (ex: `https://analysis.windows.net/powerbi/api/Dataset.Read.All`), or the resource default if permissions are pre-configured for the app (ex: `https://analysis.windows.net/powerbi/api/.default`). This may also include OpenID scopes. |
| refresh_token |required |The refresh_token that you acquired during the token request. |
| redirect_uri |required |The same redirect_uri value that was used to acquire the authorization_code. |
| client_secret |required for web apps |The application secret that you created in the app registration portal for your app.  It should not be used in a native app, because client_secrets cannot be reliably stored on devices.  It is required for web apps and web APIs, which have the ability to store the client_secret securely on the server side. |

### Response
A successful token response will look similar to the following.

```
{
    "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6Ik5HVEZ2ZEstZnl0aEV1Q...",
    "token_type": "Bearer",
    "expires_in": 3599,
    "scope": "https://analysis.windows.net/powerbi/api/Dataset.Read.All",
    "refresh_token": "AwABAAAAvPM1KaPlrEqdFSBzjqfTGAMxZGUTdM0t4B4...",
}
```
| Parameter | Description |
| --- | --- |
| access_token |The requested access token. The app can use this token in calls to Microsoft Graph. |
| token_type |Indicates the token type value. The only type that Azure AD supports is Bearer |
| expires_in |How long the access token is valid (in seconds). |
| scope |The permissions (scopes) that the access_token is valid for. |
| refresh_token |A new OAuth 2.0 refresh token. You should replace the old refresh token with this newly acquired refresh token to ensure your refresh tokens remain valid for as long as possible. |

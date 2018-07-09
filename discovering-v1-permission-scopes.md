# Discovering permission scopes strings for Azure AD v1 applications.

In this topic, we will describe the steps to discover permission scope strings of Azure AD v1.0 resources, with the purpose of integrating their APIs in an Azure AD v2.0 application. You can also reference many [popular Azure AD v1.0 resource](popular-v1-permissions.md).

>  Note: the steps outlined in this topic are a temporary solution for locating scope strings. The developer experience between the Azure AD v1.0 and v2.0 endpoints will improve dramatically over time and not longer require hunting for scope strings.

The following steps can be performed to locate the correct permission scope strings of a Azure AD v1.0 resource.

1. Create Azure AD v1.0 application with desired permissions
2. Copy the permissions from the v1.0 application to your v2.0 application
3. Perform a code authorization flow with your v2.0 application using the resource default scope
4. Get scope names from the token response

## 1. Create Azure AD v1.0 application with desired permissions

Start by creating a v1.0 application through the Azure Portal with the desired API permission to other applications.

### Create an Azure AD v1.0 application

In the Azure Portal, create an Azure AD v1.0 application. To make a new v1.0 application, go to Azure Active Directory -> App Registrations -> New Application Registration. This is just a dummy application so don't worry about using a proper name or url.

### Remove the default scopes

Your goal is to isolate the scopes you want to use in your application. So start off with removing whatever scopes this new v1.0 application came with.

The v1.0 scopes are stored in the v1.0 application's manifest file. In the application, choose Manifest, which is next to Settings and a json file will get pulled up. In that file is a section called `requiredResourceAccess` which is a list of objects. This contains all permission scopes included in that application, organized by which application those scopes pertain to (PowerBI, Flow, etc) since each application has its own scopes. Remove all items in this list if there are any, so you will know which are the scopes you are adding yourself in the next step. This way, you make sure not to overflow your app permissions with scopes you don't even intent to use.

### Add the permission scope(s) you want to use

Back in the application, choose Settings -> Required Permissions. This is where you are specifying the kinds of permissions this simulation v1.0 application will accept. To cover whatever application you're trying to actually use, choose Add, and then select the service you're trying to hit and choose the permission scope you will want to use. For example, if you choose the PowerBI API, you can choose the `View All Groups` permissions scope. When you choose "Done," the scopes you chose will be added in the manifest file you saw earlier.

## 2. Copy the permissions from the v1.0 application to your v2.0 application

### Find the scope from the v1.0 application's manifest

Now go back to the manifest file to see the new scopes you just added. The `resourceAppId` is the application id for the service you are trying to hit (like PowerBI), and the `resourceAccess` holds the scopes included from that service (like `View All Groups`). Each object in this list has an `id` and a `type`, where the id is the scope's AAD object id <!-- confirm this -->and the `type` is `Scope.` If you added multiple scopes and/or multiple different APIs in the last step, make sure to understand this structure so you add the scopes properly in the next step.

If this is the first scope you're adding to your application manifest file, grab the entire `requiredResourceAccess` section. If it's the first scope you're adding from a new service you haven't accessed before, grab the whole object in that section including the `resourceAppId` and the `resourceAccess` sections. If you're adding a new scope but have already used this service in your application, identify which object in the `resourceAccess` list is new and only grab that.

Below is what the scope looks like for PowerBI's Read All Groups

```
"requiredResourceAccess": [
    {
        "resourceAppId": "00000009-0000-0000-c000-000000000000",
        "resourceAccess": [
            {
                "id": "47df08d3-85e6-4bd3-8c77-680fbe28162e",
                "type": "Scope"
            }
        ]
    }
]
```

### Add the scope to your v2.0 application's manifest

Now that you grabbed the v1.0 scope manifest section, you have to add it to your actual Azure AD v2.0 application's manifest file. Go to wherever you registered your Azure AD v2.0 application and edit its manifest file. Paste in the subsection you determined from the previous step, based on what already exists in your application. For example, don't paste in the entire `requiredResourceAccess` section if you have already have it.

## 3. Perform a code authorization flow with your v2.0 application using the resource default scope

Now when building your HTTP request to hit this service's endpoint from your Azure AD v2.0 application, the scope you just preconfigured in your v2.0 application will be accessible under the resource default scope.

### Authorization Request

This is similar to the main instructions, but you will not specify a scope and will instead use the predetermined scopes, accessed through the `/.default` scope of each of the services.

>In these examples, we use a dummy client id (6731de76-14a6-49ae-97bc-6eba6914391e) that you can use for testing as well if you'd like.

```
// Line breaks for legibility only

https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
?client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=code
&redirect_uri=http%3A%2F%2Flocalhost%2Fmyapp%2F
&response_mode=query
&scope=https%3A%2F%2Fanalysis.windows.net%2Fpowerbi%2Fapi%2F.default
&state=12345
```
| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  The allowed values are `common` for both Microsoft accounts and work or school accounts, `organizations` for work or school accounts only, `consumers` for Microsoft accounts only, and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| response_type |required |Must include `code` for the authorization code flow. |
| redirect_uri |recommended |The redirect_uri of your app, where authentication responses can be sent and received by your app.  It must exactly match one of the redirect_uris you registered in the app registration portal, except it must be URL encoded.  For native and mobile apps, you should use the default value of `https://login.microsoftonline.com/common/oauth2/nativeclient`. |
| scope |required |A space-separated list of the Microsoft Graph permissions that you want the user to consent to. This may also include OpenID scopes. |
| response_mode |recommended |Specifies the method that should be used to send the resulting token back to your app.  Can be `query` or `form_post`. |
| state |recommended |A value included in the request that will also be returned in the token response.  It can be a string of any content that you wish.  A randomly generated unique value is typically used for [preventing cross-site request forgery attacks](http://tools.ietf.org/html/rfc6749#section-10.12).  The state is also used to encode information about the user's state in the app before the authentication request occurred, such as the page or view they were on. |

> **Important**: Microsoft Graph exposes two kinds of permissions: application and delegated. For apps that run with a signed-in user, you request delegated permissions in the `scope` parameter. These permissions delegate the privileges of the signed-in user to your app, allowing it to act as the signed-in user when making calls to Microsoft Graph. For more detailed information about the permissions available through Microsoft Graph, see the [Permissions reference](./permissions_reference.md).

### Authorization Response

The response from this GET request will be a query string with a `code` parameter which you will extract to use in the next step.

```
// Line breaks for legibility only

http://localhost/myapp/
?code={...}
&state=12345
&session_state={...}
```

## 4. Get scope names from the token response

Now that you have authenticated with a code flow, you can use your code from the last step's response to get an access token response.

### Token Request

Use the below HTTP/1.1 POST request to request an access token. Again, this sample shows our dummy client id and its matching client secret.

<!-- The rest of the docs use this format for the first line: "POST /organizations/oauth2/v2.0/token HTTP/1.1"

AND for the second line they add https:// to the beginning, which breaks when I run it in Postman

AND for the URIs they use encoded, which break the calls when I run them in postman. -->
```
POST https://login.microsoftonline.com/organizations/oauth2/v2.0/token
Host: login.microsoftonline.com
Content-Type: application/x-www-form-urlencoded

client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&scope=https://analysis.windows.net/powerbi/api/.default
&code=`{insert from previous step}`
&redirect_uri=http://localhost/myapp/
&grant_type=authorization_code
&client_secret=JqQX2PNo9bpM0uEihUPzyrh    // NOTE: Only required for web apps
```
| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  The allowed values are `common` for both Microsoft accounts and work or school accounts, `organizations` for work or school accounts only, `consumers` for Microsoft accounts only, and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| grant_type |required |Must be `authorization_code` for the authorization code flow. |
| scope |required |A space-separated list of scopes.  The scopes requested in this leg must be equivalent to or a subset of the scopes requested in the first (authorization) leg.  If the scopes specified in this request span multiple resource servers, then the v2.0 endpoint will return a token for the resource specified in the first scope. |
| code |required |The authorization_code that you acquired in the first leg of the flow. |
| redirect_uri |required |The same redirect_uri value that was used to acquire the authorization_code. |
| client_secret |required for web apps |The application secret that you created in the app registration portal for your app.  It should not be used in a native app, because client_secrets cannot be reliably stored on devices.  It is required for web apps and web APIs, which have the ability to store the client_secret securely on the server side. |

### Token Response

Unlike usual, you don't actually care about the access token right now. You just need to look at the `scopes` parameter from the response. Now you have the names of all the v1.0 scopes you wanted, so you can use those on your v2.0 application. In this sample response, the scope we wanted was Dataset.Read.All within the PowerBI API.

```
{
    "token_type": "Bearer",
    "scope": "https://analysis.windows.net/powerbi/api/Dataset.Read.All https://analysis.windows.net/powerbi/api/.default",
    "expires_in": 3600,
    "ext_expires_in": 0,
    "access_token": "{...}"
}
```
# Discovering permission scopes strings for Azure AD v1 applications.

In this topic, we will describe the steps to discover permission scope strings of Azure AD v1.0 resources, with the purpose of integrating their APIs into an Azure AD v2.0 application. You can also reference many [popular Azure AD v1.0 resources](popular-v1-permissions.md).

>  Note: the steps outlined in this topic are a temporary solution for locating scope strings. The developer experience between the Azure AD v1.0 and v2.0 endpoints will improve dramatically over time and not longer require hunting for scope strings.

The following steps can be performed to locate the correct permission scope strings of a Azure AD v1.0 resource.

1. Create Azure AD v1.0 application with desired permissions
2. Copy the permissions from the v1.0 application to your v2.0 application
3. Perform a code authorization flow with your v2.0 application using the resource default scope
4. Get scope names from the token response

## 1. Create Azure AD v1.0 application with desired permissions

Start by creating a v1.0 application through the Azure Portal with the desired API permission to other applications.

### Create an Azure AD v1.0 application

In the Azure Portal, create an Azure AD v1.0 application. To make a new v1.0 application, select Azure Active Directory -> App Registrations -> New Application Registration. The details of this app registration are not important as it will only be used for extracting scope information from it's manifest. The application can be deleted after completing all of step 2.

### Remove the default scopes

New Azure AD v1.0 application registrations are provisioned with one default permission, the "Sign in and read user profile" delegated permission for the "Windows Azure Active Directory" API. You should remove this permission to help isolate the permission scopes you are trying to discover. You can remove is by selecting Settings -> Required Permissions, then selecting and deleting "Windows Azure Active Directory".

### Add the permission scope(s) you want to use

In the Required Permissions section of the application, select Add and use the "Select an API" and "Select permissions" interfaces to add the desired permissions. For example, if you search for and select the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` API, you can choose the `View all Datasets` delegated permission scope.

## 2. Copy the permissions from the v1.0 application to a v2.0 application

In this step, you will copy the permissions configured in the Azure AD v1.0 application and configure them in the Azure AD v2.0 application by editting the application manifests.

### Find the scope from the v1.0 application's manifest

From the Azure AD v1.0 application registration in Azure portal, select the Manifest button to display the json manifest for the application. Locate and copy the entire `requiredResourceAccess` section in the manifest. The example below shows what this section looks like for an app configured with the `View all Datasets` permission of the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` resource. It is possible for this section to contain multiple resources and/or scopes.

```
"requiredResourceAccess": [
    {
        "resourceAppId": "7f33e027-4039-419b-938e-2f8ca153e68e",
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

Navigate to the application settings page of the Azure AD v2.0 application. If you haven't yet registered an Azure AD v2.0 application, follow the steps in [register your app with the Azure AD v2.0 endpoint](https://developer.microsoft.com/en-us/graph/docs/concepts/auth_register_app_v2). Locate and select the `Edit Application Manifest` button at the bottom of the page to display the manifest. Locate the `requiredResourceAccess` section of the manifest and replace it with the `requiredResourceAccess` section you copied from the Azure AD v1.0 application registration. Remember to save the changes to the manifest.

## 3. Perform a code authorization flow using the resource default scope

In this step, you will perform a manual code authorization flow against the Azure AD v2.0 endpoint using the Azure AD v1.0 resource default permission scope. This will help reveal the friendly scope strings for the Azure AD v1.0 resource when an access token is acquired.

### Authorization Request

When performing the authorization request, use the resource default scope of `{app_uri}/.default`. The example below uses the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` resource with the default scope of `https://analysis.windows.net/powerbi/api/.default`.

```
// Line breaks for legibility only
// Note the use of the "organizations" tenant

https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize
?client_id=6731de76-14a6-49ae-97bc-6eba6914391e
&response_type=code
&redirect_uri=http://localhost/myapp/
&response_mode=query
&scope=https://analysis.windows.net/powerbi/api/.default
&state=12345
```

| Parameter |  | Description |
| --- | --- | --- |
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  For Azure AD v1.0 resources, the allowed values are `organizations` and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| response_type |required |Must include `code` for the authorization code flow. |
| redirect_uri |recommended |The redirect_uri of your app, where authentication responses can be sent and received by your app.  It must exactly match one of the redirect_uris you registered in the app registration portal, except it must be URL encoded.  For native and mobile apps, you should use the default value of `https://login.microsoftonline.com/common/oauth2/nativeclient`. |
| scope |required |A space-separated list of permissions that you want the user to consent to. This can be well-known scopes such as those used by the Microsoft Graph (ex: Files.Read), a combination of app URI and scope string (ex: `https://analysis.windows.net/powerbi/api/Dataset.Read.All`), or the resource default if permissions are pre-configured for the app (ex: `https://analysis.windows.net/powerbi/api/.default`). This may also include OpenID scopes. |
| response_mode |recommended |Specifies the method that should be used to send the resulting token back to your app.  Can be `query` or `form_post`. |
| state |recommended |A value included in the request that will also be returned in the token response.  It can be a string of any content that you wish.  A randomly generated unique value is typically used for [preventing cross-site request forgery attacks](http://tools.ietf.org/html/rfc6749#section-10.12).  The state is also used to encode information about the user's state in the app before the authentication request occurred, such as the page or view they were on. |

> **Important**: Microsoft Graph exposes two kinds of permissions: application and delegated. For apps that run with a signed-in user, you request delegated permissions in the `scope` parameter. These permissions delegate the privileges of the signed-in user to your app, allowing it to act as the signed-in user when making calls to Microsoft Graph. For more detailed information about the permissions available through Microsoft Graph, see the [Permissions reference](./permissions_reference.md).

### Authorization Response

If the user consents to the permissions your app requested, the response will contain the authorization code in the `code` parameter. Here is an example of a successful response to the request above. Because the `response_mode` parameter in the request was set to `query`, the response is returned in the query string of the redirect URL.

```
GET http://localhost/myapp/?
code=M0ab92efe-b6fd-df08-87dc-2c6500a7f84d
&state=12345
```

### Token Request

Use the authorization `code` received in the previous step to request an access token by sending a `POST` request to the `/token` endpoint. Similar to the first step, you should use the resource default scope of `{app_uri}/.default`. The example below uses the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` resource with the default scope of `https://analysis.windows.net/powerbi/api/.default`.

```
// Note the use of the "organizations" tenant
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
| tenant |required |The `{tenant}` value in the path of the request can be used to control who can sign into the application.  For Azure AD v1.0 resources, the allowed values are `organizations` and tenant identifiers such as the tenant ID or domain name.  For more detail, see [protocol basics](https://docs.microsoft.com/azure/active-directory/develop/active-directory-v2-protocols#endpoints). |
| client_id |required |The Application ID that the registration portal ([apps.dev.microsoft.com](https://apps.dev.microsoft.com/?referrer=https://azure.microsoft.com/documentation/articles&deeplink=/appList)) assigned your app. |
| grant_type |required |Must be `authorization_code` for the authorization code flow. |
| scope |required |A space-separated list of permissions that you want the user to consent to. This can be well-known scopes such as those used by the Microsoft Graph (ex: Files.Read), a combination of app URI and scope string (ex: `https://analysis.windows.net/powerbi/api/Dataset.Read.All`), or the resource default if permissions are pre-configured for the app (ex: `https://analysis.windows.net/powerbi/api/.default`). This may also include OpenID scopes. |
| code |required |The authorization_code that you acquired in the first leg of the flow. |
| redirect_uri |required |The same redirect_uri value that was used to acquire the authorization_code. |
| client_secret |required for web apps |The application secret that you created in the app registration portal for your app.  It should not be used in a native app, because client_secrets cannot be reliably stored on devices.  It is required for web apps and web APIs, which have the ability to store the client_secret securely on the server side. |

## 4. Get scope names from the token response

The access token response from the code authorization flow contains a list of the permissions that the access token is good for in the `scope` parameter. These include the individual permission scope strings that can be used incrementally/dynamically with the Azure AD v2.0 endpoint. In the example response, the response includes the `View all Datasets` permission of the `PowerBI API Service (Microsoft.Azure.AnalysisServices)` resource or `https://analysis.windows.net/powerbi/api/Dataset.Read.All`.

```
{
    "token_type": "Bearer",
    "scope": "https://analysis.windows.net/powerbi/api/Dataset.Read.All https://analysis.windows.net/powerbi/api/.default",
    "expires_in": 3600,
    "ext_expires_in": 0,
    "access_token": "{...}"
}
```
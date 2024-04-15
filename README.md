# spa-app-on-behalf-of-flow
Configuration of OAuth 2.0 On-Behalf-Of flow for Single Page Web Application.

## Overview
This article describes the steps needed to configure and verify On-Behalf-Of flow for a Single Page Application (SPA) using Microsoft Entra ID.
The main purpose is to have a simple way to verify the configuration of App registrations without too much effort. The setup is not a guide for a production deployment setup.

The end user will login to the the SPA app (APP) that will call a backend API (API-01) which will then call a second API (API-02) with delegated permissions of the logged in user.
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/63557a72-9fb4-4409-9734-c0796ceaef4d)

## Prerequisites
Permission to create and configura App registrations in Microsoft Entra ID
A REST client (like Postman or VSCode REST Client) to perform HTTP Post requests 

## Configuration

### App
Create an Entra ID App Registration for the SPA App
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/ba8659bc-3fdc-414d-8fe0-276f15e86c9d)

Use the following settings:

- Redirect URI: `http://localhost`. Select platform `Single-page Application (SPA)`
- Authentication (Implicit grant and hybrid flows): `Access tokens` and `ID tokens`

### API-01
Create an Entra ID App Registration for the API-01 APP
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/3328bc94-47b6-409c-b217-a5d4a10372d2)

Use the following settings:
Expose API: `api://<api-01 client id>/user_impersonate` with consent `Admin and Users`
Authorized client apps: `<app client id>` on scope `user_impersonate`
Create a client secret

> [!NOTE]  
> The name of the scope (`user_impersonate`) is not important and can be anything.



### API-02
Create an Entra ID App Registration for the API-02 App
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/1bf0590e-3b8f-42a7-94f4-7524277488bc)

Use the following settings
Expose API: `api://<api-02 client id>/call_api` with consent `Admin and Users`
Authorized client apps: `<api-01 client id>` on scope `call_api`

> [!NOTE]  
> Again, the name of the scope (`call_api`) is not important and can be set to something that is more relevant for your application.

## Validate configuration
Login to App and call first Api
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/3aa8cc2c-2a17-4705-b524-60bd3af39010)

Do a GET request by using the following URL in your browser
```
https://login.microsoftonline.com/<tenant id>/oauth2/v2.0/authorize?
client_id=<app-a client id>&
response_type=id_token token&
redirect_uri=http://localhost&
scope=openid api://<api-01 client id>/user_impersonate &
response_mode=fragment&state=12345&nonce=678910
```
Change `<tenant id>`, `<client id>` and `<api-01 client id>` to match your configuration

You should then be prompted with a login dialog and after a successful login you should see an empty page. Since we don't have any web application listening on http://localhost the page will be empty but the URL will contain the result of the login. 

![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/28b4d3b8-9d2e-4bd7-870e-c46f29c37631)

Copy the access token from the URL (make sure to remove all other url parameters before and after the access token)
Validate the token by decoding the JWT using a tool such as `jwt.io` or `jwt.ms`.
Ensure the following
- Audience (`aud`) should be `api://<api-01 client id>`
- Application id (`appid`) should be client id of APP
- Scope (`scp`) should be `user_impersonate`

Call second API on-behalf-of the user

![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/e9d76ffa-279b-43a5-aeb8-7d7df013b69f)

Use a REST tool to make a POST request to get a token for API-02
```
POST https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer&
client_id={{api-01 client id}}&
client_secret={{api-01 client secret}}&
assertion={{app-a access token}}&
requested_token_use=on_behalf_of&
scope={{api://<api-02 client id/call_api}}
```
Change `<tenant id>`, `<api-01 client id>`, `<api-02 client id>` and `<app a access token>` to match your configuration

> [!NOTE]  
> `app a access token` is the token received in the previous step with all additional parameters removed.

As a result, you will you will get another access token in the response.
Ensure the following
- Audience (`aud`) should be `api://<api-01 client id>`
- Application Id (`appid`) should be client id of API-01
- Scope (`scp`) should be `call_api`

## Considerations
It is technically possible to leverage only two App Registrations and merge the APP and API-01 in the same registration
![image](https://github.com/pewill-msft/spa-app-on-behalf-of-flow/assets/105436708/39f8d5c8-a109-43b3-bbf9-94cc9813465d)

See [Use of a single application](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow#use-of-a-single-application) for details

The configuration steps would be identical but instead of doing the API-01 configuration on a separate Entra ID app it is done in the same app registration. The benefits would be that only two App registrations are needed (APP and API-02) but it can be a bit harder to understand the authentication flow.

## References
[Microsoft identity platform and OAuth 2.0 On-Behalf-Of flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-on-behalf-of-flow)


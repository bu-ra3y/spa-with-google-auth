# spa-with-google-auth

An example of a Single Page App that uses Google Authentication

## Recreate this project

### Create a new Vue project using Vue CLI

Update Vue CLI

```shell
npm update @vue/cli
```

Create a new project with Vue CLI

```shell
vue create spa-with-google-auth
```

Here are the options I chose:

```text
Vue CLI v4.5.10
? Please pick a preset: Manually select features
? Check the features needed for your project: Choose Vue version, Babel, Linter
? Choose a version of Vue.js that you want to start the project with 2.x
? Pick a linter / formatter config: Prettier
? Pick additional lint features: Lint on save
? Where do you prefer placing config for Babel, ESLint, etc.? In package.json
? Save this as a preset for future projects? (y/N)
```

### Create a Google App

https://console.developers.google.com/

Create an App

Go to Google Developer Console, API Credentials: https://console.developers.google.com/apis/credentials

`Create Credentials` Button up top > `OAuth Client ID`
`Application Type` = `Web Application`
Give it a name, done.
Note the `App ID` and `App Secret`.

Note the section called `Authorized redirect URIs`.
The URIs here will need to match the URI that Cognito asks Google to redirect to. (we'll get to that below).

### Create a Cognito User Pool

https://console.aws.amazon.com/cognito/users/#/pool/new/create
Default settings

### Add Google as a Federated Identity provider

User Pool Settings > Federation > Identity Providers > Google

Input
`Google app ID`,
`App secret`, and
`Authorize scope` = `profile email openid`

Now go to User Pool Settings > Federation > Attribute Mapping > Google
Tick the Capture checkbox for Google attribute `email` and map it to User pool attribute `Email`

#### Add an App Client

Apparently this will create a Web App that will use the Cognito built-in webpages for signing in users.

User Pool Settings > General Settings > App Clients > Add an app client

Un-tick `Generate Client Secret`

Done, create it.

User Pool Settings > App Integration > App Client Settings
Tick to enable Google only (the only Auth mechanism I want to allow)
The Callback URL is where the user is directed after authenticating.
For now, to support development work:
`Callback URL` = `http://localhost:8081`

Sign out URL is ??
`Sign out URL` = `http://localhost:8081`

TODO: should I use `Authorization code grant` OAuth flow to be more secure?

Tick to allow `implicit grant` OAuth flow which returns a Cognito User Pool token to the user.
May be less secure than `Authorization code grant` which returns an authorization token
which my web app would then exchange for a Cognito user pool token using a custom backend,
never exposing the Cognito user pool token to my web app. The Cognito User Pool token is a JWT.

Tick all `Allowed OAuth Scopes`.

Now head to User Pool Settings > App Integration > Domain Name

You're choosing what the URL will look like when users are on the signup / sign-in pages.

Note the full domain name you chose.

View the sign-in/signup page by going to https://<domainName>/login?response_type=token&client_id=<cognitoAppClientID>&redirect_uri=<cognitoAppClientCallbackUrl>

`response_type` should be

- `code` for Authorization Code Grant flow
- `token` for Implicit Code Grant flow

In my case:
https://socotra.auth.us-east-1.amazoncognito.com/login?response_type=token&client_id=3pcr6kgujtv3e4esatvlkc5ukk&redirect_uri=http://localhost:8081

After Authorization the browser heads to the callback URI
with the granted token appended to the URI like this:

<redirectUri>/#id_token=1234&expires_in=3600&token_type=Bearer

In my case I am using the `Implicit Code Grant` flow,
so the token should be a Cognito User Pool token.

### Add the Cognito App Client as an allowed redirect for the Google App

Go back to your Google API Credentials: https://console.developers.google.com/apis/credentials
Click on the `OAth 2.0 Client ID` you created, `Authorized redirect URIs` section
and add:
https://<cognitoAppClientDomainYouChose>/oauth2/idpresponse
In my case:
https://socotra.auth.us-east-1.amazoncognito.com/oauth2/idpresponse

Save

### Set up API Gateway to use Cognito User Pool as the Authorizer for APIs

Create an API on API Gatway

Create an Authorizer that uses your Cognito User Pool
TODO: API Gateway > <my API> > Authorizers > ...
name: google_auth

### Create an endpoint on API Gateway that uses the Authorizor

Idea here is a simple test of the Authorizer

Create a basic resource on the API to test that the Cognito Pool Authorizor works
API Gateway > <my API> > Resources > / > Actions > Create Resource
`Resource Name`: `authorized`
`Resource path`: `/authorized`
Enable API Gateway CORS: yes

Add a method to it:
API Gateway > <my API> > Resources > /authorized > Actions > Create Method > POST
Integration Type: Mock
Method Request > Authorization: google_auth (the API Gateway Authorizor you set up)

#### Enable CORS on API Gateway

A helpful primer here: https://www.serverless.com/blog/cors-api-gateway-survival-guide

This will allow me to make the calls the my APIs on API Gateway from my webapp which is running on a different domain.

API Gateway > <my API> > Resources > /authorized > Actions > Enable CORS

This will allow requests to this Resource (`/authorized`). The methods on this Resource are using the Mock Integration
meaning API Gateway will do the responding instead of, say, a Lambda function.
If I was using a Lambda function to serve an endpoint, I'd need to include in the response from my Lambda,
the necessary CORS headers.

API Gateway > <my API> > Gateway Responses > Unauthorized > Edit > Reponse Headers > add this:
`Access-Control-Allow-Origin` = `'*'`

This will allow an Unauthorized answer to go through with a normal `401`.

### Test Authentication

Deploy your API
API Gateway > <my API> > Resources > Actions > Deploy API

And test:

Authenticate by visiting the Cognito User Pool App Client UI: https://socotra.auth.us-east-1.amazoncognito.com/login?response_type=token&client_id=3pcr6kgujtv3e4esatvlkc5ukk&redirect_uri=http://localhost:8081
Take the `id_token` that is returned as a URL fragment.
Send it as the content of the `Authorization` header in a `POST` to your `/authorized` endpoint
like mine: https://5xhos1hzad.execute-api.us-east-1.amazonaws.com/dev/authorized
A reply of `200` means that the token is good; `401` otherwise.

### Create an endpoint that gives you the secret data

...but only if you are authorized!

Create another resource in your API, `/secrets` with method `GET`.
Again, set it to use the Authorizor,
In the method's Integration Response > 200 response > Mapping Templates > application/json > use this JSON for the template:

```json
{
  "data": [1, 2, 3, 4, 5]
}
```

Enable CORS on the Resource: API Gateway > <my API> > Resources > /secrets > Actions > Enable CORS
Include the options to configure CORS on the `Gateway Responses`.

Deploy the API again (Actions > Deploy API)

Now `GET` the `/secrets` endpoint,
like mine: https://5xhos1hzad.execute-api.us-east-1.amazonaws.com/dev/secrets
With no Authorization header you should get a `401`.
With an expired token you should get a `401` with a message mentions the expiration.
With a good token you'll get a `200` and you'll see the data come back.

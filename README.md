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
The URIs here will need to match those used in your Cognito User Pool App Client Redirect URIs (see below).
For now input: `http://localhost:8081`
Note: if you are changing this setting, it may take time to go into effect (minutes?).

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
Tick to enable Cognito User Pool and Google
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

<template>
  <v-container>
    <v-row class="text-center">
      <v-col cols="12">
        <v-img
          :src="require('../assets/logo.svg')"
          class="my-3"
          contain
          height="200"
        />
      </v-col>

      <v-col class="mb-4">
        <h1 class="display-2 font-weight-bold mb-3">
          Welcome!
        </h1>
        <div v-if="token">
          <h2>You are authenticated!</h2>
          <span v-if="secretData">Secret Data: {{ secretData }} </span>
          <span v-else>"Getting secret data..."</span>
        </div>
        <div v-else>
          <v-btn block color="primary" elevation="8" large :href="loginUrl">
            Authenticate
          </v-btn>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
const axios = require("axios");
const host = "socotra.auth.us-east-1.amazoncognito.com";
const redirectURI = "http://localhost:8081";
const clientId = "3pcr6kgujtv3e4esatvlkc5ukk"; // ID of Cognito Client App that will log in users to my Cognito User Pool

export default {
  name: "HelloWorldPkce",

  data: function() {
    return {
      loginUrl: "",
      token: null,
      secretData: null
    };
  },
  methods: {
    authenticate: async function(authorizationCode) {
      // Auth Step 3: request tokens
      console.debug(`Auth Step 3: Requesting Tokens`);
      const codeVerifier = sessionStorage.getItem("pkceCodeVerifier"); // load PKCE Code Verifier from session storage
      // const query = `grant_type=authorization_code&client_id=${clientId}&code_verifier=${codeVerifier}&redirect_uri=${redirectURI}&code=${authorizationCode}`;
      const parameters = new URLSearchParams();
      parameters.append("grant_type", "authorization_code");
      parameters.append("client_id", clientId);
      parameters.append("code_verifier", codeVerifier);
      parameters.append("redirect_uri", redirectURI);
      parameters.append("code", authorizationCode);
      try {
        const response = await axios.post(
          `https://${host}/oauth2/token`,
          parameters,
          {
            headers: { "Content-Type": "application/x-www-form-urlencoded" }
          }
        );
        const {
          id_token
          // access_token,
          // refresh_token,
          // expires_in,
          // token_type
        } = response.data;
        this.token = id_token;
        this.loadProtectedContent();
      } catch (e) {
        console.error("Failed to get Tokens");
        console.error(e);
      }
    },
    getRandomString: () => {
      const randomItems = new Uint32Array(28);
      crypto.getRandomValues(randomItems); // Fill the array with random numbers
      const binaryStringItems = randomItems.map(
        dec => `0${dec.toString(16).substr(-2)}`
      );
      return binaryStringItems.reduce((acc, item) => `${acc}${item}`, "");
    },
    createCodeChallengeAndSetLoginUrl: async function(state, codeVerifier) {
      // Create PKCE code challenge
      const arrayHash = await this.encryptStringWithSHA256(codeVerifier);
      const codeChallenge = this.hashToBase64url(arrayHash);
      sessionStorage.setItem("pkceCodeChallenge", codeChallenge);

      // Set Login URL
      this.setLoginUrl(state, codeChallenge);
    },
    setLoginUrl: function(state, codeChallenge) {
      if (!codeChallenge || !state) {
        // Make sure we have these
        console.error(
          `Could not set Login URL because I'm missing either a PKCE State (${state}) or a PKCE Code Challenge (${codeChallenge})`
        );
      } else {
        const path = "oauth2/authorize";
        const query = `response_type=code&state=${state}&client_id=${clientId}&redirect_uri=${redirectURI}&scope=openid&code_challenge_method=S256&code_challenge=${codeChallenge}`;
        this.loginUrl = `https://${host}/${path}?${query}`;
      }
    },
    encryptStringWithSHA256: async str => {
      const PROTOCOL = "SHA-256";
      const textEncoder = new TextEncoder();
      const encodedData = textEncoder.encode(str);
      return crypto.subtle.digest(PROTOCOL, encodedData); // returns a Promise
    },
    hashToBase64url(arrayBuffer) {
      const items = new Uint8Array(arrayBuffer);
      const stringifiedArrayHash = items.reduce(
        (acc, i) => `${acc}${String.fromCharCode(i)}`,
        ""
      );
      const decodedHash = btoa(stringifiedArrayHash);
      // Return Base-64 encoded URL
      return decodedHash
        .replace(/\+/g, "-")
        .replace(/\//g, "_")
        .replace(/=+$/, "");
    },
    loadProtectedContent: function() {
      this.getSecretData();
    },
    getSecretData: async function() {
      try {
        const response = await axios.get(
          "https://5xhos1hzad.execute-api.us-east-1.amazonaws.com/dev/secrets",
          { headers: { Authorization: this.token } }
        );
        this.secretData = response.data.data;
      } catch (e) {
        console.debug(`Failed to get secret data`);
        console.debug(e);
        this.badTokens();
      }
    },
    badTokens: function() {
      this.token = null;
    }
  },
  created: function() {
    // Check to see if we are receiving an Authorization Code in the Browser's URL bar
    // FIXME: does not work if I have to actually log in to my google account in browser
    // FIXME: auth flow does not work if: go through auth flow then do page refresh
    const urlQueryString = window.location.search.substring(1); // remove leading "?"
    let urlSearchParams = new URLSearchParams(urlQueryString);
    const code = urlSearchParams.get("code");
    const state = urlSearchParams.get("state");
    if (code && state) {
      // Auth Step 2: User has logged in and we are receiving the resulting Authorization Code and PKCE State
      console.debug(`Auth Step 2: Received Authorization Code + State`);
      // Remove query string from Browser URL
      const newUrl = window.location
        .toString()
        .replace(window.location.search, "");
      window.history.replaceState({}, document.title, newUrl);

      // Verify that the PKCE State matches
      const priorPkceState = sessionStorage.getItem("pkceState");
      //  PKCE State is created before we request the Authorization Code
      //   and then used after we receive the Authorization Code when the App is loaded a second time
      //   in order to verify against the State sent alongside the Authorization Code
      if (state !== priorPkceState) {
        console.error(
          `PKCE state that I have (${priorPkceState}) does not match that
          returned with the Authorization Code (${state}). Authorization error.`
        );
        // TODO: try again to restart the Auth process
      } else {
        this.authenticate(code); // Move to the next Auth step
      }
    } else {
      // Auth Step 1: Auth flow not started
      //  signified by not receiving a code in the URL
      console.debug(`Auth Step 1: Log in!`);
      // signified by we are neither receiving a code nor have one stored
      // redirect to Cognito OAuth2 /oath2/authorize for sign-in (via the Authorize button shown)

      // Create PKCE "pkceState" and save it in Session storage
      //  This is something created before we request the Authorization Code
      //   and then used again after we receive the code when the App is loaded a second time
      const state = this.getRandomString();
      sessionStorage.setItem("pkceState", state);

      // Create PKCE code verifier
      const codeVerifier = this.getRandomString();
      sessionStorage.setItem("pkceCodeVerifier", codeVerifier);

      // Next step is to create PKCE Code Challenge
      //  but it must be done async
      //  therefore we will initiate the process
      //   and the Login button will appear when
      //   the PKCE Code Challenge has been created
      this.createCodeChallengeAndSetLoginUrl(state, codeVerifier);
    }
  }
};
</script>

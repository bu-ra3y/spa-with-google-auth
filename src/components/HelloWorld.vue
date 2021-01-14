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
        <div v-if="!token">
          <v-btn
            block
            color="primary"
            elevation="8"
            large
            href="https://socotra.auth.us-east-1.amazoncognito.com/login?response_type=token&client_id=3pcr6kgujtv3e4esatvlkc5ukk&redirect_uri=http://localhost:8081"
            >Authenticate</v-btn
          >
        </div>
        <div v-else>
          <h2>You have a {{ goodToken ? "GOOD" : "" }} token!</h2>
          <span v-if="secretData">Secret Data: {{ secretData }}</span>
        </div>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
const axios = require("axios");

export default {
  name: "HelloWorld",

  data: function() {
    return {
      token: null,
      goodToken: null,
      secretData: null
    };
  },
  methods: {
    testToken: async function() {
      try {
        await axios.post(
          "https://5xhos1hzad.execute-api.us-east-1.amazonaws.com/dev/authorized",
          {},
          { headers: { Authorization: this.token } }
        );
        this.goodToken = true; // Token is good
        this.loadProtectedContent();
      } catch (e) {
        // Token is not good
        this.token = null;
      }
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
      this.goodToken = false;
    }
  },
  created: function() {
    // We need to get authentication information from the URL Fragments provided
    //  AWS Cognito is going to ba passing Authentication tokens
    //  via URL Fragments AKA Hash (not URL Query Parameters)
    const fragments = window.location.hash.substring(1); // remove leading "#"
    const params = new URLSearchParams(fragments); // The Fragments are in the same format as Query/Search Params
    this.token = params.get("id_token");
    if (this.token) {
      this.testToken();
    }
  }
};
</script>

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
        <h2>
          We {{ haveAuthInformation ? "DO" : "do NOT" }} have auth information
          for you.
        </h2>
        <ul v-if="haveAuthInformation">
          <li>id_token: {{ id_token }}</li>
          <li>access_token: {{ access_token }}</li>
          <li>token_type: {{ token_type }}</li>
          <li>expires_in: {{ expires_in }}</li>
        </ul>
        <v-btn
          v-else
          block
          color="primary"
          elevation="8"
          large
          href="https://socotra.auth.us-east-1.amazoncognito.com/login?response_type=token&client_id=3pcr6kgujtv3e4esatvlkc5ukk&redirect_uri=http://localhost:8081"
          >Authenticate</v-btn
        >
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
export default {
  name: "HelloWorld",

  data: function() {
    return {
      id_token: null,
      access_token: null,
      expires_in: null,
      token_type: null
    };
  },
  computed: {
    haveAuthInformation: function() {
      return (
        // must have all
        !!this.id_token &&
        !!this.access_token &&
        !!this.expires_in &&
        !!this.token_type
      );
    }
  },
  created: function() {
    // We need to get authentication information from the URL Fragments provided
    //  AWS Cognito is going to ba passing Authentication tokens
    //  via URL Fragments AKA Hash (not URL Query Parameters)
    const fragments = window.location.hash.substring(1); // remove leading "#"
    const params = new URLSearchParams(fragments); // The Fragments are in the same format as Query/Search Params
    console.debug(`Fragments provided:`);
    for (const [key, value] of params.entries()) {
      console.debug(`${key}=${value}`);
    }
    this.id_token = params.get("id_token");
    this.access_token = params.get("access_token");
    this.expires_in = params.get("expires_in");
    this.token_type = params.get("token_type");
  }
};
</script>

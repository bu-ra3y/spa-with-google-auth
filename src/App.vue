<template>
  <div id="app">
    <img alt="Vue logo" src="./assets/logo.png" />
    <h1>Welcome!</h1>
    <h2>
      We {{ haveAuthInformation ? "DO" : "do NOT" }} have auth information for
      you.
    </h2>
    <ul v-if="haveAuthInformation">
      <li>id_token: {{ id_token }}</li>
      <li>access_token: {{ access_token }}</li>
      <li>token_type: {{ token_type }}</li>
      <li>expires_in: {{ expires_in }}</li>
    </ul>

    <HelloWorld />
  </div>
</template>

<script>
import HelloWorld from "./components/HelloWorld.vue";

export default {
  name: "App",
  components: {
    HelloWorld
  },
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

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>

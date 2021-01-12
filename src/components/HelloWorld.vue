<template>
  <div class="hello">
    <h1>Welcome!</h1>
    <h2>{{ msg }}</h2>
    <h3>URL Parameters: {{ window.location.search }}</h3>
  </div>
</template>

<script>
export default {
  name: "HelloWorld",
  data: function() {
    return {
      msg: "You are NOT authenticated."
    };
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>

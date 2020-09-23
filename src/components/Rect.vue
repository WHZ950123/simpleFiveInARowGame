<template>
  <div>
    <router-link to="/">hello</router-link>
	<div class="rec"></div>
  </div>
</template>

<script>
export default {
  name: 'Rect',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App'
    }
  }
}
</script>

<style scoped>
.rec {
	width: 100px;
	height: 100px;
	background-color: #eee;
}
</style>

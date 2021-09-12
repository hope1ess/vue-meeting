<template>
  <div>
    hello world
    <button @click="goToMeeting">go to meeting</button>
  </div>
</template>

<script>
export default {
  name: 'Home',

  data() {
    return {

    }
  },

  methods: {
    goToMeeting() {
      this.$router.push('/meeting');
    }
  }
}
</script>

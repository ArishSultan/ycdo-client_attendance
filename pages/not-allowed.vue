<template>
  <div>
    <h1>
      You are Not allowed to view this application, Only admin has access to
      this application
    </h1>
    <v-btn color="primary" @click="$auth.logout()">Log Out</v-btn>
  </div>
</template>

<script>
export default {
  name: 'NotAllowed',
}
</script>

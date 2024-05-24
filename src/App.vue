<template>
  <v-app>
    <v-main>
      <Main/>
      <app-alert/>
    </v-main>
  </v-app>
</template>

<script>
import Main from '@/components/Main.vue'
import AppAlert from "@/components/AppAlert"

export default {
  name: 'App',

  components: {
    Main, AppAlert,
  },

  data: () => ({
    //
  }),
}
</script>

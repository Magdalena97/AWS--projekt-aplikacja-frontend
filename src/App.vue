<template>
  <v-app>
    <v-app-bar 
      app
      color="primary"
      dark
    >
      <div class="d-flex align-center">
        <v-img
          alt="Vuetify Logo"
          class="shrink mx-2"
          contain
          src="https://icons.iconarchive.com/icons/uiconstock/socialmedia/128/AWS-icon.png"
          transition="scale-transition"
          width="40"
        />

        <h1>AWS Sender</h1>
      </div>

      <v-spacer></v-spacer>

    </v-app-bar>

    <v-main>
      
      <Form />

    </v-main>
  </v-app>
</template>

<script>
import Form from './components/Form';

export default {
  name: 'App',

  components: {
    Form,
  },

  data: () => ({
    //
  }),
};
</script>

<template>
  <v-app>
    <v-main>
      <router-view />
    </v-main>
  </v-app>
</template>

<script>
import "./assets/style.css";

export default {
  name: 'App',
};
</script>
<style>
/* 스크롤바 웹킷 엔진 */
::-webkit-scrollbar {
    width: 10px;
    height: 8px;
    background: #ffffff;
    /* display: none; */
}
::-webkit-scrollbar-thumb {
    border-radius: 3.5px;
    background-color: #dedeeb;
}
::-webkit-scrollbar-track {
    background: #ffffff;
}
</style>
<template>
  <div id="app">
    <ViewerComponent :style="{width: '500px', height: '300px'}"></ViewerComponent>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import ViewerComponent from './components/Viewer.vue';

@Component({
  components: {
    ViewerComponent,
  },
})
export default class App extends Vue {}
</script>



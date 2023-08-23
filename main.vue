<template xmlns:v-slot=http://www.w3.org/1999/XSL/Transform>
  <div>
    <v-app id="inspire">
      <bwc-website microsite>
        <bosch-header></bosch-header>
        <v-main class="main__wrapper">
          <div style="margin: auto; padding: 30px;">
            <router-view></router-view>
          </div>
        </v-main>
        <bosch-footer></bosch-footer>
      </bwc-website>
    </v-app>
  </div>
</template>

<script>
import { heartbeat } from '@/api/heatbeatApi';
import BoschFooter from '../components/BoschFooter.vue';
import BoschHeader from './BoschHeader.vue';

export default {
  name: 'Main',
  components: {
    BoschHeader,
    BoschFooter,
  },
  watch: {
    $route(newRouter) {
      this.$store.state.currentPath = newRouter.path;
    },
  },
  data: () => ({}),
  methods: {},
  created() {
    setInterval(() => {
      if (localStorage.getItem('x-token')) {
        heartbeat()
          .then(() => {
          });
      }
    }, 10000);
  },
  props: {
    source: String,
  },
};
</script>

<style scoped>
</style>

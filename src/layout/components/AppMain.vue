<template>
  <section id="appMain" class="app-main">
    <transition name="fade-transform" mode="out-in">
      <keep-alive :include="cachedViews">
        <router-view :key="key" />
      </keep-alive>
    </transition>
  </section>
</template>

<script>
export default {
  name: 'AppMain',
  computed: {
    cachedViews() {
      return this.$store.state.tagsView.cachedViews
    },
    key() {
      return this.$route.path
    },
    openRoute() {
      return this.$store.state.utils.openRoute
    }
  },
  created() {
    this.readRouteParameters()
  },
  methods: {
    readRouteParameters() {
      if (this.$store.getters.getIsLoadedOpenRoute) {
        return
      }
      this.$store.dispatch('setOpenRoute', {
        path: this.$route.path,
        name: this.$route.name,
        fullPath: this.$route.fullPath,
        params: {
          ...this.$route.params
        },
        query: {
          ...this.$route.query
        },
        isLoaded: true
      })
    }
  }
}
</script>

<style lang="scss" scoped>
.app-main {
  /* 50= navbar  50  */
  min-height: calc(100vh - 50px);
  width: 100%;
  position: relative;
  overflow: hidden;
}

.fixed-header+.app-main {
  padding-top: 50px;
  height: 100vh;
  overflow: auto;
}

.hasTagsView {
  .app-main {
    /* 84 = navbar + tags-view = 50 + 34 */
    min-height: calc(100vh - 84px);
  }

  .fixed-header+.app-main {
    padding-top: 84px;
  }
}
</style>

<style lang="scss">
// fix css style bug in open el-dialog
.el-popup-parent--hidden {
  .fixed-header {
    padding-right: 15px;
  }
}
</style>

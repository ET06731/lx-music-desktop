<template>
  <div id="my-list" :class="$style.container">
    <MyList v-if="!listId" :active-id="lastListId" />
    <DetailView v-else ref="detailView" :list-id="listId" @back="handleBackToOverview" />
  </div>
</template>

<script>
import { getListPrevSelectId } from '@renderer/utils/data'

import MyList from './MyList/index.vue'
import DetailView from './DetailView.vue'

export default {
  name: 'List',
  components: {
    MyList,
    DetailView,
  },
  beforeRouteUpdate(to, from) {
    this.syncRoute(to, from)
    if (to.query.updated) return
    if (to.query.id != null && to.query.scrollIndex != null) {
      return {
        path: '/list',
        query: {
          id: to.query.id,
          updated: true,
        },
      }
    }
  },
  beforeRouteLeave() {
    this.$refs.detailView?.saveListPosition?.()
  },
  data() {
    return {
      listId: null,
      lastListId: null,
    }
  },
  created() {
    this.syncRoute(this.$route)
    if (!this.lastListId) {
      void getListPrevSelectId().then(id => {
        if (!this.lastListId) this.lastListId = id
      })
    }
  },
  methods: {
    syncRoute(route, from) {
      const id = typeof route.query.id == 'string' ? route.query.id : null
      this.listId = id
      if (id) this.lastListId = id
      if (id && route.query.scrollIndex != null) {
        const isAnimation = from?.query.id == route.query.id
        this.$nextTick(() => {
          this.$refs.detailView?.handleRestoreScroll?.(route.query.scrollIndex, isAnimation)
        })
      }
    },
    handleBackToOverview() {
      void this.$router.replace({ path: '/list' })
    },
  },
}
</script>

<style lang="less" module>
@import '@renderer/assets/styles/layout.less';

.container {
  overflow: hidden;
  height: 100%;
  display: flex;
  position: relative;
}

</style>

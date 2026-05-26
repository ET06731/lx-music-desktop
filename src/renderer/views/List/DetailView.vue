<template>
  <div :class="$style.container">
    <div :class="$style.header">
      <div :class="$style.coverWrap">
        <img v-if="cover" :class="$style.cover" :src="cover" loading="lazy" decoding="async">
        <div v-else :class="$style.coverFallback">
          <svg-icon name="music" />
        </div>
      </div>
      <div :class="$style.info">
        <div :class="$style.badges">
          <span :class="$style.badge">{{ $t('my_list') }}</span>
          <span v-if="sourceLabel" :class="$style.badgeSecondary">{{ sourceLabel }}</span>
        </div>
        <h2 :class="$style.title">{{ displayName }}</h2>
        <div :class="$style.meta">
          <span :class="$style.metaItem">
            <svg-icon name="music" />
            {{ musicTotal }}
          </span>
          <span v-if="listUpdateTimes[listId]" :class="$style.metaText">{{ listUpdateTimes[listId] }}</span>
          <span v-else-if="subText" :class="$style.metaText">{{ subText }}</span>
        </div>
      </div>
      <base-btn :class="$style.backBtn" outline min @click="$emit('back')">{{ $t('back') }}</base-btn>
    </div>
    <div :class="$style.listWrap">
      <MusicList ref="musicList" :list-id="listId" />
    </div>
  </div>
</template>

<script>
import { computed, onBeforeUnmount, ref, watch } from '@common/utils/vueTools'
import { defaultList, loveList, userLists, listUpdateTimes } from '@renderer/store/list/state'
import { getListMusics } from '@renderer/store/list/action'
import { sourceNames } from '@renderer/store'
import MusicList from './MusicList/index.vue'

export default {
  name: 'ListDetailView',
  components: {
    MusicList,
  },
  props: {
    listId: {
      type: String,
      required: true,
    },
  },
  emits: ['back'],
  setup(props, { emit }) {
    const musicList = ref(null)
    const musics = ref([])

    const listInfo = computed(() => {
      if (props.listId == defaultList.id) return defaultList
      if (props.listId == loveList.id) return loveList
      return userLists.find(item => item.id == props.listId) ?? null
    })
    const displayName = computed(() => {
      if (!listInfo.value) return ''
      if (listInfo.value.id == defaultList.id || listInfo.value.id == loveList.id) return window.i18n.t(listInfo.value.name)
      return listInfo.value.name
    })
    const sourceLabel = computed(() => {
      const source = listInfo.value?.source
      return source ? sourceNames.value[source] : ''
    })
    const cover = computed(() => {
      for (const item of musics.value) {
        if (item.meta.picUrl) return item.meta.picUrl
      }
      return ''
    })
    const musicTotal = computed(() => musics.value.length)
    const subText = computed(() => {
      const firstMusic = musics.value[0]
      if (firstMusic?.singer) return firstMusic.singer
      return sourceLabel.value
    })

    const loadList = async() => {
      if (!listInfo.value) {
        emit('back')
        return
      }
      const targetId = props.listId
      const list = await getListMusics(props.listId)
      if (props.listId != targetId) return
      musics.value = [...list]
    }

    const handleMyListUpdate = (ids) => {
      if (!ids.includes(props.listId)) return
      void loadList()
    }

    watch(() => props.listId, () => {
      void loadList()
    }, {
      immediate: true,
    })

    watch(() => userLists.map(item => item.id).join('|'), () => {
      if (props.listId == defaultList.id || props.listId == loveList.id) return
      if (!userLists.some(item => item.id == props.listId)) emit('back')
    })

    window.app_event.on('myListUpdate', handleMyListUpdate)

    onBeforeUnmount(() => {
      window.app_event.off('myListUpdate', handleMyListUpdate)
    })

    return {
      musicList,
      listUpdateTimes,
      displayName,
      sourceLabel,
      cover,
      musicTotal,
      subText,
    }
  },
  methods: {
    handleRestoreScroll(scrollIndex, isAnimation) {
      this.$refs.musicList?.handleRestoreScroll?.(scrollIndex, isAnimation)
    },
    saveListPosition() {
      this.$refs.musicList?.saveListPosition?.()
    },
  },
}
</script>

<style lang="less" module>
@import '@renderer/assets/styles/layout.less';

.container {
  height: 100%;
  width: 100%;
  display: flex;
  flex-flow: column nowrap;
  min-width: 0;
}

.header {
  flex: none;
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px 18px 14px;
  border-bottom: var(--color-list-header-border-bottom);
}

.coverWrap {
  flex: none;
  width: 84px;
  height: 84px;
  border-radius: 14px;
  overflow: hidden;
  background: linear-gradient(135deg, var(--color-primary-background-hover), var(--color-primary-background-active));
  box-shadow: 0 10px 24px rgba(0, 0, 0, .18);
}

.cover,
.coverFallback {
  width: 100%;
  height: 100%;
}

.cover {
  display: block;
  object-fit: cover;
}

.coverFallback {
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--color-font-label);

  :global(svg) {
    width: 30px;
    height: 30px;
  }
}

.info {
  flex: auto;
  min-width: 0;
}

.badges {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}

.badge,
.badgeSecondary {
  display: inline-flex;
  align-items: center;
  height: 22px;
  padding: 0 10px;
  border-radius: 999px;
  font-size: 12px;
  line-height: 1;
}

.badge {
  color: var(--color-primary);
  background-color: var(--color-primary-font-active);
}

.badgeSecondary {
  color: var(--color-font-label);
  background-color: var(--color-primary-background-hover);
}

.title {
  font-size: 22px;
  line-height: 1.2;
  color: var(--color-font);
  .mixin-ellipsis-1();
}

.meta {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-top: 10px;
  color: var(--color-font-label);
  font-size: 12px;
}

.metaItem {
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.metaText {
  .mixin-ellipsis-1();
}

.backBtn {
  flex: none;
}

.listWrap {
  min-height: 0;
  flex: auto;
  display: flex;
}

</style>

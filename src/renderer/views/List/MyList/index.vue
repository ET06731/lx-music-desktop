<template>
  <div ref="dom_lists" :class="$style.container">
    <div :class="$style.listHeader">
      <h2 :class="$style.listsTitle">{{ $t('my_list') }}</h2>
      <div :class="$style.headerBtns">
        <button :class="$style.headerBtn" :aria-label="$t('lists__new_list_btn')" @click="isShowNewList = true">
          <svg version="1.1" xmlns="http://www.w3.org/2000/svg" xlink="http://www.w3.org/1999/xlink" height="70%" viewBox="0 0 24 24" space="preserve">
            <use xlink:href="#icon-list-add" />
          </svg>
        </button>
        <button :class="$style.headerBtn" :aria-label="$t('list_update_modal__title')" @click="isShowListUpdateModal = true">
          <svg version="1.1" xmlns="http://www.w3.org/2000/svg" xlink="http://www.w3.org/1999/xlink" style="transform: rotate(45deg);" height="70%" viewBox="0 0 24 24" space="preserve">
            <use xlink:href="#icon-refresh" />
          </svg>
        </button>
      </div>
    </div>

    <div ref="dom_lists_list" class="scroll" :class="[$style.listsContent, { [$style.sortable]: isModDown }]">
      <div
        class="default-list"
        :class="[$style.listsItem, { [$style.active]: activeId == defaultList.id }, { [$style.clicked]: rightClickItemIndex == -2 }, { [$style.fetching]: fetchingListStatus[defaultList.id] }]"
        :aria-label="$t(defaultList.name)"
        :aria-selected="activeId == defaultList.id"
        @contextmenu="handleListsItemRigthClick($event, -2)"
      >
        <div :class="$style.cardMain" @click="handleListToggle(defaultList.id)">
          <div :class="$style.cardCover">
            <img v-if="getCardInfo(defaultList.id).cover" :class="$style.coverImg" :src="getCardInfo(defaultList.id).cover" loading="lazy" decoding="async">
            <div v-else :class="$style.coverFallback">
              <svg-icon name="music" />
            </div>
          </div>
          <div :class="$style.cardDesc">
            <h3 :class="$style.cardTitle">{{ $t(defaultList.name) }}</h3>
            <p :class="$style.cardSubTitle">{{ getCardInfo(defaultList.id).subTitle }}</p>
            <div :class="$style.cardMeta">
              <span :class="$style.metaItem">
                <svg-icon name="music" />
                {{ getCardInfo(defaultList.id).total }}
              </span>
              <span v-if="getCardInfo(defaultList.id).metaText" :class="$style.metaText">{{ getCardInfo(defaultList.id).metaText }}</span>
            </div>
          </div>
        </div>
      </div>

      <div
        class="default-list"
        :class="[$style.listsItem, { [$style.active]: activeId == loveList.id }, { [$style.clicked]: rightClickItemIndex == -1 }, { [$style.fetching]: fetchingListStatus[loveList.id] }]"
        :aria-label="$t(loveList.name)"
        :aria-selected="activeId == loveList.id"
        @contextmenu="handleListsItemRigthClick($event, -1)"
      >
        <div :class="$style.cardMain" @click="handleListToggle(loveList.id)">
          <div :class="$style.cardCover">
            <img v-if="getCardInfo(loveList.id).cover" :class="$style.coverImg" :src="getCardInfo(loveList.id).cover" loading="lazy" decoding="async">
            <div v-else :class="$style.coverFallback">
              <svg-icon name="music" />
            </div>
          </div>
          <div :class="$style.cardDesc">
            <h3 :class="$style.cardTitle">{{ $t(loveList.name) }}</h3>
            <p :class="$style.cardSubTitle">{{ getCardInfo(loveList.id).subTitle }}</p>
            <div :class="$style.cardMeta">
              <span :class="$style.metaItem">
                <svg-icon name="music" />
                {{ getCardInfo(loveList.id).total }}
              </span>
              <span v-if="getCardInfo(loveList.id).metaText" :class="$style.metaText">{{ getCardInfo(loveList.id).metaText }}</span>
            </div>
          </div>
        </div>
      </div>

      <div
        v-for="(item, index) in userLists"
        :key="item.id"
        class="user-list"
        :class="[$style.listsItem, { [$style.active]: activeId == item.id }, { [$style.clicked]: rightClickItemIndex == index }, { [$style.fetching]: fetchingListStatus[item.id] }]"
        :data-index="index"
        :aria-label="item.name"
        :aria-selected="activeId == item.id"
        @contextmenu="handleListsItemRigthClick($event, index)"
      >
        <div :class="$style.cardMain" @click="handleListToggle(item.id)">
          <div :class="$style.cardCover">
            <img v-if="getCardInfo(item.id).cover" :class="$style.coverImg" :src="getCardInfo(item.id).cover" loading="lazy" decoding="async">
            <div v-else :class="$style.coverFallback">
              <svg-icon name="music" />
            </div>
          </div>
          <div :class="$style.cardDesc">
            <h3 :class="$style.cardTitle">{{ item.name }}</h3>
            <p :class="$style.cardSubTitle">{{ getCardInfo(item.id).subTitle }}</p>
            <div :class="$style.cardMeta">
              <span :class="$style.metaItem">
                <svg-icon name="music" />
                {{ getCardInfo(item.id).total }}
              </span>
              <span v-if="getCardInfo(item.id).metaText" :class="$style.metaText">{{ getCardInfo(item.id).metaText }}</span>
            </div>
          </div>
        </div>
        <base-input
          :class="$style.listsInput"
          type="text"
          :value="item.name"
          :placeholder="item.name"
          @click.stop
          @keyup.enter="handleSaveListName(index, $event)"
          @blur="handleSaveListName(index, $event)"
        />
      </div>

      <transition enter-active-class="animated-fast slideInLeft" leave-active-class="animated-fast fadeOut" @after-leave="isNewListLeave = false" @after-enter="$refs.dom_listsNewInput.focus()">
        <div v-if="isShowNewList" :class="[$style.listsItem, $style.listsNew, { [$style.newLeave]: isNewListLeave }]">
          <div :class="$style.newCard">
            <div :class="$style.coverFallback">
              <svg-icon name="music" />
            </div>
            <base-input
              ref="dom_listsNewInput"
              :class="$style.listsInput"
              type="text"
              :placeholder="$t('lists__new_list_input')"
              @keyup.enter="handleCreateList"
              @blur="handleCreateList"
            />
          </div>
        </div>
      </transition>
    </div>

    <base-menu v-model="isShowMenu" :menus="menus" :xy="menuLocation" item-name="name" @menu-click="handleMenuClick" />
    <DuplicateMusicModal v-model:visible="isShowDuplicateMusicModal" :list-info="duplicateListInfo" />
    <ListSortModal v-model:visible="isShowListSortModal" :list-info="sortListInfo" />
    <ListUpdateModal v-model:visible="isShowListUpdateModal" />
  </div>
</template>

<script>
import { openUrl } from '@common/utils/electron'

import musicSdk from '@renderer/utils/musicSdk'
import DuplicateMusicModal from './components/DuplicateMusicModal.vue'
import ListSortModal from './components/ListSortModal.vue'
import ListUpdateModal from './components/ListUpdateModal.vue'

import { defaultList, loveList, userLists, fetchingListStatus, listUpdateTimes } from '@renderer/store/list/state'
import { removeUserList, getListMusics } from '@renderer/store/list/action'

import { onBeforeUnmount, reactive, ref, watch } from '@common/utils/vueTools'
import { useRouter } from '@common/utils/vueRouter'

import { dialog } from '@renderer/plugins/Dialog'
import { sourceNames } from '@renderer/store'

import { saveListPrevSelectId } from '@renderer/utils/data'

import { useI18n } from '@renderer/plugins/i18n'


import useShare from './useShare'
import useMenu from './useMenu'
import useListUpdate from './useListUpdate'
import useSort from './useSort'
import useDarg from './useDarg'
import useEditList from './useEditList'
import useDuplicate from './useDuplicate'

const emptyCardInfo = Object.freeze({
  cover: '',
  total: 0,
  subTitle: '',
  metaText: '',
})

export default {
  name: 'MyLists',
  components: {
    DuplicateMusicModal,
    ListSortModal,
    ListUpdateModal,
  },
  props: {
    activeId: {
      type: String,
      default: '',
    },
  },
  emits: ['show-menu'],
  setup(props, { emit }) {
    const router = useRouter()
    const t = useI18n()

    const dom_lists_list = ref(null)
    const rightClickItemIndex = ref(-10)
    const cardInfoMap = reactive({})

    const { handleImportList, handleExportList } = useShare()
    const { isShowListUpdateModal, handleUpdateSourceList } = useListUpdate()
    const { isShowListSortModal, sortListInfo, handleSortList } = useSort()
    const { isShowDuplicateMusicModal, duplicateListInfo, handleDuplicateList } = useDuplicate()
    const { handleRename, handleSaveListName, isShowNewList, isNewListLeave, handleCreateList } = useEditList({ dom_lists_list })

    let refreshCardInfoMark = 0
    const refreshCardInfo = async() => {
      const mark = ++refreshCardInfoMark
      const lists = [defaultList, loveList, ...userLists]
      const musicLists = await Promise.all(lists.map(async list => ({
        id: list.id,
        list: await getListMusics(list.id),
      })))
      if (mark != refreshCardInfoMark) return

      for (const item of lists) {
        const currentMusics = musicLists.find(list => list.id == item.id)?.list ?? []
        const firstMusic = currentMusics[0]
        const cover = currentMusics.find(music => music.meta.picUrl)?.meta.picUrl ?? ''
        const subTitle = item.source
          ? sourceNames.value[item.source]
          : firstMusic?.singer || t('my_list')
        const metaText = listUpdateTimes[item.id] || firstMusic?.meta.albumName || ''
        cardInfoMap[item.id] = {
          cover,
          total: currentMusics.length,
          subTitle,
          metaText,
        }
      }
    }

    const getCardInfo = (id) => cardInfoMap[id] ?? emptyCardInfo

    const handleOpenSourceDetailPage = async(listInfo) => {
      const { source, sourceListId } = listInfo
      if (!sourceListId) return
      let url
      if (/board__/.test(sourceListId)) {
        const id = sourceListId.replace(/board__/, '')
        url = musicSdk[source].leaderboard.getDetailPageUrl(id)
      } else if (musicSdk[source]?.songList?.getDetailPageUrl) {
        url = await musicSdk[source].songList.getDetailPageUrl(sourceListId)
      }
      if (!url) return
      void openUrl(url)
    }

    const handleRemove = (listInfo) => {
      void dialog.confirm({
        message: t('lists__remove_tip', { name: listInfo.name }),
        confirmButtonText: t('lists__remove_tip_button'),
      }).then(isRemove => {
        if (!isRemove) return
        void removeUserList([listInfo.id])
      })
    }

    const {
      menus,
      menuLocation,
      isShowMenu,
      showMenu,
      menuClick,
    } = useMenu({
      emit,

      handleImportList,
      handleExportList,
      handleUpdateSourceList,
      handleOpenSourceDetailPage,
      handleSortList,
      handleDuplicateList,
      handleRename,
      handleRemove,
    })

    const handleListsItemRigthClick = (event, index) => {
      rightClickItemIndex.value = index
      showMenu(event, index)
    }

    const handleListToggle = (id) => {
      if (isModDown.value) return
      saveListPrevSelectId(id)
      void router.push({
        path: '/list',
        query: { id },
      })
    }

    const handleMenuClick = (action) => {
      if (rightClickItemIndex.value < -2) return
      let index = rightClickItemIndex.value
      rightClickItemIndex.value = -10
      menuClick(action, index)
    }

    const { isModDown } = useDarg({ dom_lists_list, handleMenuClick, handleSaveListName })

    const handleMyListUpdate = () => {
      void refreshCardInfo()
    }

    watch(() => userLists.map(item => `${item.id}_${item.name}_${item.source ?? ''}_${item.sourceListId ?? ''}`).join('|'), () => {
      void refreshCardInfo()
    }, {
      immediate: true,
    })

    watch(() => Object.keys(listUpdateTimes).map(id => `${id}_${listUpdateTimes[id]}`).join('|'), () => {
      void refreshCardInfo()
    })

    window.app_event.on('myListUpdate', handleMyListUpdate)

    onBeforeUnmount(() => {
      window.app_event.off('myListUpdate', handleMyListUpdate)
    })

    return {
      rightClickItemIndex,
      defaultList,
      loveList,
      userLists,
      sourceNames,
      fetchingListStatus,
      dom_lists_list,
      isShowListUpdateModal,
      isShowListSortModal,
      sortListInfo,
      isShowDuplicateMusicModal,
      duplicateListInfo,
      handleSaveListName,
      isShowNewList,
      isNewListLeave,
      handleCreateList,
      handleListsItemRigthClick,
      isShowMenu,
      handleMenuClick,
      menus,
      menuLocation,
      handleListToggle,
      isModDown,
      getCardInfo,
    }
  },
}
</script>

<style lang="less" module>
@import '@renderer/assets/styles/layout.less';

.container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-flow: column nowrap;
  min-width: 0;
}

.listHeader {
  flex: none;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 16px 10px;
  border-bottom: var(--color-list-header-border-bottom);
}

.listsTitle {
  font-size: 20px;
  line-height: 1.2;
  color: var(--color-font);
}

.headerBtns {
  display: flex;
  gap: 8px;
}

.headerBtn {
  width: 34px;
  height: 34px;
  border: none;
  border-radius: 999px;
  background-color: transparent;
  color: var(--color-button-font);
  cursor: pointer;
  transition: background-color @transition-normal, opacity @transition-normal;
  opacity: .75;

  &:hover {
    opacity: 1;
    background-color: var(--color-primary-background-hover);
  }

  &:active {
    background-color: var(--color-primary-background-active);
  }

  svg {
    vertical-align: middle;
  }
}

.listsContent {
  flex: auto;
  min-width: 0;
  overflow-y: auto !important;
  padding: 14px 16px 16px;
  display: flex;
  flex-wrap: wrap;
  align-content: flex-start;
  gap: 14px 16px;

  &.sortable {
    * {
      -webkit-user-drag: element;
    }

    .listsItem {
      &:hover,
      &.active,
      &.clicked {
        background-color: transparent !important;
      }

      &.dragingItem {
        .cardMain {
          box-shadow: 0 0 0 2px var(--color-primary);
        }
      }
    }

    .listsItem:not(.default-list) {
      .cardMain {
        cursor: grab;

        &:hover {
          transform: none;
        }

        &:active {
          cursor: grabbing;
        }
      }
    }
  }
}

.listsItem {
  width: calc((100% - 32px) / 3);
  min-width: 0;
  border-radius: 18px;
  transition: opacity @transition-normal;
  background-color: transparent;

  &.active {
    .cardMain {
      box-shadow: 0 0 0 1px var(--color-primary), 0 14px 30px rgba(0, 0, 0, .18);
    }
  }

  &.clicked {
    .cardMain {
      background-color: var(--color-primary-background-hover);
    }
  }

  &.fetching {
    opacity: .55;
  }

  &.editing {
    .cardMain {
      display: none;
    }

    .listsInput {
      display: block;
    }
  }
}

.cardMain {
  display: flex;
  gap: 10px;
  min-height: 118px;
  padding: 8px;
  border-radius: 16px;
  background-color: var(--color-main-background);
  box-shadow: 0 12px 24px rgba(0, 0, 0, .12);
  cursor: pointer;
  transition: background-color @transition-normal, transform @transition-normal, box-shadow @transition-normal;

  &:hover {
    transform: translateY(-2px);
    background-color: var(--color-primary-background-hover);
    box-shadow: 0 16px 32px rgba(0, 0, 0, .18);
  }
}

.cardCover {
  position: relative;
  flex: none;
  width: 112px;
  min-width: 112px;
  aspect-ratio: 1 / 1;
  border-radius: 12px;
  overflow: hidden;
  background: linear-gradient(135deg, var(--color-primary-background-hover), var(--color-primary-background-active));
}

.coverImg,
.coverFallback {
  width: 100%;
  height: 100%;
}

.coverImg {
  display: block;
  object-fit: cover;
}

.coverFallback {
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--color-font-label);

  :global(svg) {
    width: 34px;
    height: 34px;
  }
}

.cardDesc {
  flex: auto;
  min-width: 0;
  padding: 2px 4px 2px 0;
  display: flex;
  flex-flow: column nowrap;
}

.cardTitle {
  font-size: 14px;
  line-height: 1.3;
  color: var(--color-font);
  .mixin-ellipsis-2();
}

.cardSubTitle {
  margin-top: 6px;
  font-size: 12px;
  line-height: 1.4;
  color: var(--color-font-label);
  .mixin-ellipsis-1();
}

.cardMeta {
  margin-top: auto;
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 12px;
  color: var(--color-font-label);
  min-width: 0;
}

.metaItem {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  flex: none;
}

.metaText {
  .mixin-ellipsis-1();
}

.listsInput {
  width: 100%;
  height: 118px;
  padding: 0 16px;
  line-height: 118px;
  background: none !important;
  border-radius: 16px;
  font-size: 14px;
  display: none;
}

.listsNew {
  min-height: 118px;
}

.newCard {
  height: 118px;
  padding: 8px;
  border-radius: 16px;
  display: flex;
  align-items: center;
  gap: 10px;
  background-color: var(--color-primary-background-hover);

  .coverFallback {
    flex: none;
    width: 112px;
    min-width: 112px;
    border-radius: 12px;
    background: linear-gradient(135deg, var(--color-primary-background-hover), var(--color-primary-background-active));
  }

  .listsInput {
    display: block;
    height: 100%;
    line-height: 102px;
    padding-right: 8px;
    border-radius: 12px;
  }
}

.newLeave {
  margin-top: -118px;
  z-index: -1;
}

@media (max-width: 1100px) {
  .listsItem {
    width: calc((100% - 16px) / 2);
  }
}

@media (max-width: 700px) {
  .listsContent {
    padding: 12px;
    gap: 14px;
  }

  .listsItem {
    width: 100%;
    min-width: 0;
  }
}

</style>

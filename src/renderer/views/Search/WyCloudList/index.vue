<template>
  <div :class="$style.container">
    <material-online-list
      ref="listRef"
      :page="listInfo.page"
      :limit="listInfo.limit"
      :total="listInfo.total"
      :list="listInfo.list"
      :no-item="listInfo.noItemLabel"
      @toggle-page="handleTogglePage"
      @play-list="handlePlayList"
    />
  </div>
</template>

<script setup lang="ts">
import { LIST_IDS } from '@common/constants'
import { formatPlayTime } from '@common/utils/common'
import { markRawList, onBeforeUnmount, ref, watch } from '@common/utils/vueTools'
import { useRoute, useRouter } from '@common/utils/vueRouter'
import { playList } from '@renderer/core/player/action'
import { addListMusics, getListMusics } from '@renderer/store/list/action'
import { appSetting } from '@renderer/store/setting'
import { toNewMusicInfo } from '@renderer/utils'
import { getUserApiScript } from '@renderer/utils/ipc'
import { httpFetch } from '@renderer/utils/request'

interface Props {
  page: number
  keyword: string
}

interface CloudSong {
  id?: string | number
  songId?: string | number
  matchSongId?: string | number
  cloudSongId?: string | number
  name?: string
  artist?: string
  album?: string
  duration?: number
  cover?: string
  quality?: string | null
}

interface CloudResponse {
  songs?: CloudSong[]
  count?: number
}

interface HttpFetchTask<T> {
  promise: Promise<T>
  cancelHttp?: () => void
}

interface ListInfo {
  page: number
  maxPage: number
  limit: number
  total: number
  list: LX.Music.MusicInfo[]
  noItemLabel: string
}

const props = defineProps<Props>()
const router = useRouter()
const route = useRoute()

const listRef = ref<any>(null)
const listInfo = ref<ListInfo>({
  page: 1,
  maxPage: 0,
  limit: 30,
  total: 0,
  list: [],
  noItemLabel: '',
})

let requestObj: HttpFetchTask<{ body: CloudResponse }> | null = null

const clearList = (message = '') => {
  listInfo.value.list = []
  listInfo.value.total = 0
  listInfo.value.maxPage = 0
  listInfo.value.page = props.page
  listInfo.value.noItemLabel = message
}

const unescapeJsString = (value: string) => value
  .replace(/\\\\/g, '\\')
  .replace(/\\'/g, '\'')
  .replace(/\\"/g, '"')
  .replace(/\\r/g, '\r')
  .replace(/\\n/g, '\n')

const matchConstString = (script: string, name: string) => {
  const result = new RegExp(`(?:const|let|var)\\s+${name}\\s*=\\s*(['\"])((?:\\\\.|(?!\\1).)*)\\1`).exec(script)
  return result ? unescapeJsString(result[2]) : ''
}

const parseBridgeConfig = (script: string) => {
  const apiBase = matchConstString(script, 'API_BASE')
  const token = matchConstString(script, 'ACCESS_TOKEN')
  if (!apiBase || !token) throw new Error('missing bridge config')

  return {
    apiBase,
    token,
  }
}

const buildUrl = (base: string, pathname: string, params: Record<string, string | number>) => {
  const url = new URL(pathname, base)
  for (const [key, value] of Object.entries(params)) {
    if (value === '') continue
    url.searchParams.set(key, String(value))
  }
  return url.toString()
}

const buildQualityInfo = (quality?: string | null) => {
  const types: LX.Music.MusicQualityType[] = []
  const _types: LX.Music._MusicQualityType = {}
  const addType = (type: LX.Quality) => {
    types.push({ type, size: null })
    _types[type] = { size: null }
  }

  addType('128k')
  if (quality === 'exhigh' || quality === 'lossless' || quality === 'hires') addType('320k')
  if (quality === 'lossless' || quality === 'hires') addType('flac')
  if (quality === 'hires') addType('flac24bit')

  return { types, _types }
}

const normalizeSong = (song: CloudSong) => {
  const songId = String(song.matchSongId ?? song.songId ?? song.id ?? '')
  const { types, _types } = buildQualityInfo(song.quality)
  const musicInfo = toNewMusicInfo({
    source: 'wy',
    songmid: songId,
    name: song.name ?? '',
    singer: song.artist ?? '',
    interval: song.duration ? formatPlayTime(song.duration / 1000) : '--/--',
    albumName: song.album ?? '',
    img: song.cover ?? '',
    types,
    _types,
    albumId: '',
  })
  if (song.cloudSongId != null) {
    musicInfo.meta ??= {}
    ;(musicInfo.meta as LX.Music.MusicInfo['meta'] & { cloudSongId?: string }).cloudSongId = String(song.cloudSongId)
  }
  return musicInfo
}

const loadList = async() => {
  requestObj?.cancelHttp?.()
  listInfo.value.page = props.page

  if (!/^user_api/.test(appSetting['common.apiSource'])) {
    clearList(window.i18n.t('wy_cloud__source_required'))
    return Promise.resolve()
  }

  if (props.keyword.trim()) {
    clearList(window.i18n.t('wy_cloud__search_not_supported'))
    return Promise.resolve()
  }

  listInfo.value.noItemLabel = window.i18n.t('list__loading')

  return getUserApiScript(appSetting['common.apiSource']).then(async script => {
    if (!script) {
      clearList(window.i18n.t('wy_cloud__source_required'))
      return null
    }

    const { apiBase, token } = parseBridgeConfig(script)
    requestObj = httpFetch(buildUrl(apiBase, '/cloud/list', {
      token,
      limit: listInfo.value.limit,
      offset: (props.page - 1) * listInfo.value.limit,
    }), {
      method: 'get',
    }) as unknown as HttpFetchTask<{ body: CloudResponse }>
    return await requestObj.promise
  }).then(response => {
    if (!response) return
    const data = response.body ?? {}
    const list = Array.isArray(data.songs) ? markRawList(data.songs.map(normalizeSong)) : []

    listInfo.value.list = list
    listInfo.value.total = Number(data.count) || 0
    listInfo.value.maxPage = Math.max(1, Math.ceil(listInfo.value.total / listInfo.value.limit))
    listInfo.value.noItemLabel = list.length ? '' : window.i18n.t('no_item')

    if (list.length) {
      setTimeout(() => {
        listRef.value?.scrollToTop?.()
      })
    }
  }).catch((error: any) => {
    console.log(error)
    clearList(error?.message === 'missing bridge config'
      ? window.i18n.t('wy_cloud__script_invalid')
      : window.i18n.t('list__load_failed'))
  })
}

watch(() => [props.page, props.keyword, appSetting['common.apiSource']], () => {
  void loadList()
}, {
  immediate: true,
})

onBeforeUnmount(() => {
  requestObj?.cancelHttp?.()
})

const handleTogglePage = (page: number) => {
  void router.replace({
    path: route.path,
    query: {
      ...route.query,
      page,
    },
  })
}

const handlePlayList = async(index: number) => {
  const targetSong = listInfo.value.list[index]
  if (!targetSong) return

  await addListMusics(LIST_IDS.DEFAULT, [targetSong])
  const defaultListMusics = await getListMusics(LIST_IDS.DEFAULT)
  const targetIndex = defaultListMusics.findIndex(song => song.id === targetSong.id)
  if (targetIndex > -1) playList(LIST_IDS.DEFAULT, targetIndex)
}
</script>

<style lang="less" module>
.container {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}
</style>

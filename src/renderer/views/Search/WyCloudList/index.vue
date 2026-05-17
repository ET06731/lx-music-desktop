<template>
  <div :class="$style.container">
    <material-online-list
      ref="listRef"
      :page="listInfo.page"
      :limit="listInfo.limit"
      :total="listInfo.total"
      :list="listInfo.list"
      :no-item="listInfo.noItemLabel"
      check-api-source
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

interface CloudLyricFields {
  lyric?: string
  lrc?: string
  tlyric?: string
  tlrc?: string
  rlyric?: string
  rlrc?: string
  lxlyric?: string
  rawlrc?: string
}

interface CloudSong extends CloudLyricFields {
  id?: string | number
  songId?: string | number
  matchSongId?: string | number
  cloudSongId?: string | number
  name?: string
  artist?: string
  album?: string
  duration?: number
  cover?: string
  picUrl?: string
  quality?: string | null
  lyrics?: CloudLyricFields
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

type CloudMusicMeta = LX.Music.MusicInfo['meta'] & {
  cloudSongId?: string
  picUrl?: string
  cloudLyricInfo?: {
    lyric: string
    tlyric: string
    rlyric: string
    lxlyric: string
  }
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

const CLOUD_SEARCH_LIMIT = 1000

let requestObj: HttpFetchTask<{ body: CloudResponse }> | null = null
let loadId = 0
let cloudCache: CloudSong[] = []
let cloudCacheApiSource = ''

const clearList = (message = '') => {
  listInfo.value.list = []
  listInfo.value.total = 0
  listInfo.value.maxPage = 0
  listInfo.value.page = props.page
  listInfo.value.noItemLabel = message
}

const unescapeSingleQuotedJsString = (value: string) => value
  .replace(/\\\\/g, '\\')
  .replace(/\\'/g, '\'')
  .replace(/\\r/g, '\r')
  .replace(/\\n/g, '\n')

const parseBridgeConfig = (script: string) => {
  const apiBaseMatch = script.match(/const API_BASE = '((?:\\.|[^'])*)'/)
  const tokenMatch = script.match(/const ACCESS_TOKEN = '((?:\\.|[^'])*)'/)
  if (!apiBaseMatch || !tokenMatch) throw new Error('missing bridge config')

  return {
    apiBase: unescapeSingleQuotedJsString(apiBaseMatch[1]),
    token: unescapeSingleQuotedJsString(tokenMatch[1]),
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

const firstString = (...values: Array<string | undefined | null>) => values.find(value => typeof value == 'string' && value.trim()) ?? ''

const normalizeCloudLyricInfo = (song: CloudSong) => {
  const lyric = firstString(song.lyric, song.lrc, song.lyrics?.lyric, song.lyrics?.lrc)
  const tlyric = firstString(song.tlyric, song.tlrc, song.lyrics?.tlyric, song.lyrics?.tlrc)
  const rlyric = firstString(song.rlyric, song.rlrc, song.lyrics?.rlyric, song.lyrics?.rlrc)
  const lxlyric = firstString(song.lxlyric, song.lyrics?.lxlyric)

  if (!lyric && !tlyric && !rlyric && !lxlyric) return null

  return {
    lyric,
    tlyric,
    rlyric,
    lxlyric,
  }
}

const normalizeSong = (song: CloudSong) => {
  const songId = String(song.matchSongId ?? song.songId ?? song.id ?? '')
  const picUrl = song.cover ?? song.picUrl ?? ''
  const { types, _types } = buildQualityInfo(song.quality)
  const musicInfo = toNewMusicInfo({
    source: 'wy',
    songmid: songId,
    name: song.name ?? '',
    singer: song.artist ?? '',
    interval: song.duration ? formatPlayTime(song.duration / 1000) : '--/--',
    albumName: song.album ?? '',
    img: picUrl,
    types,
    _types,
    albumId: '',
  })
  const cloudMeta = musicInfo.meta as CloudMusicMeta
  if (song.cloudSongId != null) cloudMeta.cloudSongId = String(song.cloudSongId)
  if (picUrl) cloudMeta.picUrl = picUrl

  const cloudLyricInfo = normalizeCloudLyricInfo(song)
  if (cloudLyricInfo) cloudMeta.cloudLyricInfo = cloudLyricInfo

  return musicInfo
}

const normalizeKeyword = (text?: string | number | null) => String(text ?? '')
  .toLowerCase()
  .replace(/\s+/g, '')

const isMatchedSong = (song: CloudSong, keyword: string) => {
  const target = [
    song.name,
    song.artist,
    song.album,
    song.id,
    song.songId,
    song.matchSongId,
    song.cloudSongId,
  ].map(normalizeKeyword).join('|')

  return target.includes(keyword)
}

const requestCloudList = async(apiBase: string, token: string, limit: number, offset: number) => {
  requestObj = httpFetch(buildUrl(apiBase, '/cloud/list', {
    token,
    limit,
    offset,
  }), {
    method: 'get',
  }) as unknown as HttpFetchTask<{ body: CloudResponse }>

  const response = await requestObj.promise
  return response.body ?? {}
}

const loadAllCloudSongs = async(apiBase: string, token: string, currentLoadId: number) => {
  const cacheKey = appSetting['common.apiSource']

  if (cloudCacheApiSource === cacheKey && cloudCache.length) return cloudCache

  const firstData = await requestCloudList(apiBase, token, CLOUD_SEARCH_LIMIT, 0)
  if (currentLoadId !== loadId) return []

  const songs = Array.isArray(firstData.songs) ? [...firstData.songs] : []
  const total = Number(firstData.count) || songs.length

  for (let offset = songs.length; offset < total; offset += CLOUD_SEARCH_LIMIT) {
    const data = await requestCloudList(apiBase, token, CLOUD_SEARCH_LIMIT, offset)
    if (currentLoadId !== loadId) return []

    if (Array.isArray(data.songs) && data.songs.length) songs.push(...data.songs)
    else break
  }

  cloudCache = songs
  cloudCacheApiSource = cacheKey

  return songs
}

const updateListInfo = (songs: CloudSong[], total: number) => {
  const start = (props.page - 1) * listInfo.value.limit
  const pageSongs = songs.slice(start, start + listInfo.value.limit)
  const list = markRawList(pageSongs.map(normalizeSong))

  listInfo.value.list = list
  listInfo.value.total = total
  listInfo.value.maxPage = Math.max(1, Math.ceil(total / listInfo.value.limit))
  listInfo.value.noItemLabel = list.length ? '' : window.i18n.t('no_item')

  if (list.length) {
    setTimeout(() => {
      listRef.value?.scrollToTop?.()
    })
  }
}

const loadList = async() => {
  const currentLoadId = ++loadId

  requestObj?.cancelHttp?.()
  listInfo.value.page = props.page

  if (!/^user_api/.test(appSetting['common.apiSource'])) {
    clearList(window.i18n.t('wy_cloud__source_required'))
    return Promise.resolve()
  }

  const keyword = normalizeKeyword(props.keyword.trim())

  listInfo.value.noItemLabel = window.i18n.t('list__loading')

  return getUserApiScript(appSetting['common.apiSource']).then(async script => {
    if (!script) {
      clearList(window.i18n.t('wy_cloud__source_required'))
      return null
    }

    const { apiBase, token } = parseBridgeConfig(script)

    if (keyword) {
      const allSongs = await loadAllCloudSongs(apiBase, token, currentLoadId)
      if (currentLoadId !== loadId) return null

      const filteredSongs = allSongs.filter(song => isMatchedSong(song, keyword))
      updateListInfo(filteredSongs, filteredSongs.length)

      return null
    }

    const songsData = await requestCloudList(
      apiBase,
      token,
      listInfo.value.limit,
      (props.page - 1) * listInfo.value.limit,
    )

    if (currentLoadId !== loadId) return null

    return songsData
  }).then(data => {
    if (!data) return

    const songs = Array.isArray(data.songs) ? data.songs : []
    const total = Number(data.count) || 0

    updateListInfo(songs, total)
  }).catch((error: any) => {
    if (currentLoadId !== loadId) return

    console.log(error)
    clearList(error?.message === 'missing bridge config'
      ? window.i18n.t('wy_cloud__script_invalid')
      : window.i18n.t('list__load_failed'))
  })
}

watch(() => appSetting['common.apiSource'], () => {
  cloudCache = []
  cloudCacheApiSource = ''
})

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

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'
import MusicRatingList from './components/MusicRatingList.vue'

type TrackDTO = {
  id: number | string
  title: string
  artist?: string
  user_rating?: number
}

type RatingsMap = Record<string | number, number>

const tracks = ref<TrackDTO[]>([])
const ratings = ref<RatingsMap>({})

const ctrl = new AbortController()

const clamp = (v: number, min: number, max: number) => Math.min(Math.max(Math.trunc(v), min), max)

const mergeRatingsFromServer = (serverTracks: TrackDTO[], maxStars = 5): RatingsMap => {
  const next: RatingsMap = {}
  for (const t of serverTracks) {
    const fromServer = Number.isFinite(t.user_rating as number) ? Number(t.user_rating) : undefined
    const existing = ratings.value[t.id]
    const base = fromServer ?? existing ?? 0
    next[t.id] = clamp(base, 0, maxStars)
  }
  return next
}

const loadTracks = async () => {
  const res = await fetch('/api/tracks', { method: 'GET', cache: 'no-store', signal: ctrl.signal })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  const data = (await res.json()) as TrackDTO[]
  tracks.value = data
  ratings.value = mergeRatingsFromServer(data)
}

const onRate = async (p: { id: number | string; rating: number }) => {
  ratings.value = { ...ratings.value, [p.id]: p.rating }
  const res = await fetch(`/api/ratings/${encodeURIComponent(String(p.id))}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ rating: p.rating }),
    cache: 'no-store',
    signal: ctrl.signal,
  })
  if (!res.ok) await loadTracks()
}

const refresh = async () => { await loadTracks() }

onMounted(async () => {
  try { await loadTracks() } catch (e) { console.error(e) }
})

onBeforeUnmount(() => ctrl.abort())
</script>

<template>
  <div class="page">
    <div class="toolbar">
      <button type="button" class="btn" @click="refresh">Обновить список</button>
    </div>

    <MusicRatingList
        v-model="ratings"
        :tracks="tracks"
        :max-stars="5"
        @rate="onRate"
    />
  </div>
</template>

<style scoped>
.page { display: grid; gap: 16px; padding: 16px }
.toolbar { display: flex; justify-content: flex-end }
.btn { padding: 8px 14px; border: 1px solid #d1d5db; border-radius: 6px; background: #fff; cursor: pointer }
.btn:hover { background: #f9fafb }
</style>

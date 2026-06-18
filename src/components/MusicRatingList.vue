<template>
  <section class="music-rating-list">
    <header class="list-header">
      <h2>Музыка</h2>
      <div class="legend" aria-hidden="true">Сортировка: по рейтингу ↓</div>
    </header>

    <ul class="tracks" role="list">
      <li v-for="t in sortedTracks" :key="'track-' + t.id" class="track">
        <div class="meta">
          <div class="title">{{ t.title }}</div>
          <div v-if="t.artist" class="artist">{{ t.artist }}</div>
        </div>

        <div
            class="stars"
            :aria-label="`Рейтинг: ${r(t.id)} из ${maxStars}`"
            role="slider"
            :aria-valuemin="0"
            :aria-valuemax="maxStars"
            :aria-valuenow="r(t.id)"
            tabindex="0"
            @keydown="onKey(t.id, $event)"
        >
          <button
              v-for="n in maxStars"
              :key="n"
              type="button"
              class="star"
              :class="{ filled: n <= r(t.id), readonly }"
              :disabled="readonly"
              :aria-pressed="n <= r(t.id)"
              @click="setRating(t.id, n)"
          >
            ★
          </button>

          <button
              v-if="!readonly && r(t.id) > 0"
              type="button"
              class="clear"
              @click="setRating(t.id, 0)"
              aria-label="Сбросить рейтинг"
              title="Сбросить рейтинг"
          >
            ×
          </button>
        </div>
      </li>
    </ul>
  </section>
</template>

<script setup lang="ts">
import { computed, reactive, watch, defineProps, defineEmits, onBeforeUnmount } from 'vue'

type Track = { id: string | number; title: string; artist?: string; rating?: number }

const props = defineProps<{
  tracks: Track[]
  maxStars?: number
  readonly?: boolean
  modelValue?: Record<string | number, number>
}>()

const emit = defineEmits<{
  (e: 'update:modelValue', value: Record<string | number, number>): void
  (e: 'rate', payload: { id: string | number; rating: number }): void
}>()

const state = reactive({
  ratings: {} as Record<string | number, number>
})

const maxStars = computed(() => props.maxStars ?? 5)
const readonly = computed(() => props.readonly ?? false)

const clamp = (v: number, min: number, max: number) => Math.min(Math.max(Math.trunc(v), min), max)
const r = (id: string | number) => state.ratings[id] ?? 0

const rebuildFromProps = () => {
  const next: Record<string | number, number> = {}
  const mv = props.modelValue
  for (const t of props.tracks ?? []) {
    const base = mv && Number.isFinite(mv[t.id] as number)
        ? Number(mv[t.id])
        : Number(t.rating ?? 0)
    next[t.id] = clamp(base, 0, maxStars.value)
  }
  state.ratings = next
}

// без deep/watchEffect, чтобы исключить лишние реактивные циклы
watch(
    () => [props.tracks, props.modelValue, maxStars.value],
    rebuildFromProps,
    { immediate: true }
)

const setRating = (id: string | number, value: number) => {
  if (readonly.value) return
  const next = { ...state.ratings, [id]: clamp(value, 0, maxStars.value) }
  state.ratings = next
  emit('update:modelValue', next)
  emit('rate', { id, rating: next[id] })
}

const onKey = (id: string | number, e: KeyboardEvent) => {
  if (readonly.value) return
  const cur = r(id)
  if (e.key === 'ArrowRight') { e.preventDefault(); setRating(id, clamp(cur + 1, 0, maxStars.value)) }
  else if (e.key === 'ArrowLeft') { e.preventDefault(); setRating(id, clamp(cur - 1, 0, maxStars.value)) }
  else if (e.key === 'Home') { e.preventDefault(); setRating(id, 0) }
  else if (e.key === 'End') { e.preventDefault(); setRating(id, maxStars.value) }
  else if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); setRating(id, cur === 0 ? 1 : cur) }
}

const sortedTracks = computed<Track[]>(() => {
  const withR = (props.tracks ?? []).map(t => ({ ...t, _r: r(t.id) }))
  return withR
      .sort((a, b) => (b._r - a._r) || String(a.title).localeCompare(String(b.title)))
      .map(({ _r, ...rest }) => rest)
})

onBeforeUnmount(() => {
  // нет подписок, но оставлено для симметрии в случае будущих addEventListener
})
</script>

<style scoped>
.music-rating-list { display: grid; gap: 12px; max-width: 720px; margin: 0 auto; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif }
.list-header { display: flex; align-items: baseline; justify-content: space-between }
.list-header h2 { margin: 0; font-size: 20px; font-weight: 600 }
.legend { color: #667085; font-size: 12px }
.tracks { list-style: none; padding: 0; margin: 0; display: grid; gap: 10px }
.track { display: grid; grid-template-columns: 1fr auto; align-items: center; gap: 8px; padding: 10px 12px; border: 1px solid #e5e7eb; border-radius: 10px; background: #fff }
.meta { display: grid; gap: 2px }
.title { font-size: 14px; font-weight: 600; color: #111827 }
.artist { font-size: 12px; color: #6b7280 }
.stars { display: inline-flex; align-items: center; gap: 2px; outline: none }
.star { width: 28px; height: 28px; line-height: 26px; font-size: 20px; border-radius: 6px; border: 1px solid #e5e7eb; background: #f9fafb; color: #d1d5db; cursor: pointer; display: inline-flex; align-items: center; justify-content: center; transition: background .15s ease, color .15s ease, border-color .15s ease }
.star.filled { color: #f59e0b }
.star:not(.readonly):hover { background: #fff; border-color: #cbd5e1 }
.star.readonly { cursor: default }
.clear { margin-left: 6px; width: 28px; height: 28px; border-radius: 6px; border: 1px solid #ef4444; color: #ef4444; background: #fff; cursor: pointer; font-weight: 700; line-height: 1 }
.clear:hover { background: #fef2f2 }
@media (prefers-color-scheme: dark) {
  .track { background: #0b0f14; border-color: #1f2937 }
  .title { color: #e5e7eb }
  .artist { color: #9ca3af }
  .star { background: #0b0f14; border-color: #1f2937; color: #374151 }
  .star.filled { color: #fbbf24 }
  .clear { border-color: #ef4444; color: #fecaca; background: #0b0f14 }
  .clear:hover { background: #111827 }
}
</style>

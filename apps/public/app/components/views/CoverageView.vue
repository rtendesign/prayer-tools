<script setup lang="ts">
import type { GoalProgressView } from '~~/shared/view'
import { browserTimezone } from '~/utils/commitment'

const props = defineProps<{ progress?: GoalProgressView | null, embedded?: boolean, timezone?: string }>()
const { site, link, num } = useSite()
const { t } = useI18n()
useHead(() => props.embedded ? {} : { title: `${t('coverage.title')} · ${site.value.name}` })

interface CoverageData {
  start: string
  slotMinutes: number
  timezone: string
  days: { date: string, percent: number, from: number, to: number }[]
  slots: number[]
}

const { data, error, refresh } = await useFetch<CoverageData>('/api/coverage.json', { query: { campaign: site.value.campaign?.id }, server: false })
const selectedDay = ref<string | null>(null)
const useLocal = ref(false)
const localTz = ref('UTC')
const picks = defineModel<string[]>('picks', { default: () => [] })
watch(data, (value) => {
  if (!selectedDay.value && value?.days.length) selectedDay.value = value.days.find(d => slotStart(d.to - 1).getTime() > Date.now())?.date ?? value.days[0]!.date
}, { immediate: true })

onMounted(() => {
  localTz.value = browserTimezone()
  if (props.embedded) return
  try {
    picks.value = JSON.parse(sessionStorage.getItem(`pt_picks:${site.value.orgId}`) ?? '[]')
  } catch {
    picks.value = []
  }
})

watch(picks, (v) => {
  try {
    sessionStorage.setItem(`pt_picks:${site.value.orgId}`, JSON.stringify(v))
  } catch {
    // storage blocked
  }
}, { deep: true })

const tz = computed(() => props.timezone || (useLocal.value ? localTz.value : data.value?.timezone) || 'UTC')
const showToggle = computed(() => !!data.value && localTz.value !== data.value.timezone)

const day = computed(() => data.value?.days.find(d => d.date === selectedDay.value) ?? null)
const calendarDays = computed(() => {
  const coverage = data.value
  if (!coverage) return []
  const now = Date.now()
  return coverage.days.map(d => {
    let uncovered = 0
    for (let i = d.from; i < d.to; i++) {
      if (slotStart(i).getTime() >= now && (coverage.slots[i] ?? 0) === 0) uncovered++
    }
    return { ...d, uncoveredHours: uncovered * coverage.slotMinutes / 60 }
  })
})

function slotStart(index: number): Date {
  return new Date(Date.parse(data.value!.start) + index * data.value!.slotMinutes * 60_000)
}

const slots = computed(() => {
  if (!data.value || !day.value) return []
  const fmt = new Intl.DateTimeFormat(site.value.lang, { hour: 'numeric', minute: '2-digit', timeZone: tz.value })
  const out = []
  for (let i = day.value.from; i < day.value.to; i++) {
    const at = slotStart(i)
    const iso = at.toISOString()
    const hour = Number(new Intl.DateTimeFormat('en', { hour: 'numeric', hourCycle: 'h23', timeZone: tz.value }).format(at))
    out.push({ i, iso, hour, label: fmt.format(at), count: data.value.slots[i] ?? 0, past: at.getTime() < Date.now(), picked: picks.value.includes(iso) })
  }
  return out
})
const timeGroups = computed(() => [
  { key: 'overnight', from: 0, to: 6 }, { key: 'morning', from: 6, to: 12 },
  { key: 'afternoon', from: 12, to: 18 }, { key: 'evening', from: 18, to: 24 }
].map(group => ({ ...group, slots: slots.value.filter(s => s.hour >= group.from && s.hour < group.to) })).filter(group => group.slots.length))
function pickLabel(iso: string): string {
  return new Intl.DateTimeFormat(site.value.lang, { weekday: 'short', month: 'short', day: 'numeric', hour: 'numeric', minute: '2-digit', timeZone: tz.value }).format(new Date(iso))
}

function dayLabel(date: string): string {
  return new Intl.DateTimeFormat(site.value.lang, { weekday: 'short', month: 'short', day: 'numeric', timeZone: 'UTC' }).format(new Date(`${date}T00:00:00Z`))
}

function toggle(iso: string) {
  picks.value = picks.value.includes(iso) ? picks.value.filter(p => p !== iso) : [...picks.value, iso].sort()
}

const signupLink = computed(() => `${link('signup')}?slots=${encodeURIComponent(picks.value.join(','))}`)
</script>

<template>
  <div :class="embedded ? 'min-w-0 w-full space-y-5' : 'max-w-5xl mx-auto px-4 py-10 space-y-6'">
    <header v-if="!embedded" class="space-y-3 rounded-lg bg-[#202833] text-white p-6 sm:p-8">
      <h1 class="text-3xl font-bold">{{ t('coverage.title') }}</h1>
      <p class="text-slate-200 max-w-2xl">{{ t('coverage.intro') }}</p>
    </header>
    <GoalProgress v-if="progress" :progress="progress" />

    <div v-if="!embedded && showToggle" class="flex items-center gap-3">
      <USwitch v-model="useLocal" :label="t('coverage.local_time', { tz: localTz })" />
    </div>
    <p class="text-sm text-stone-600">{{ t('coverage.times_in', { tz }) }}</p>

    <UAlert v-if="error" color="error" :title="t('coverage.load_error')" :actions="[{ label: t('coverage.retry'), onClick: () => refresh() }]" />
    <p v-else-if="!data" class="text-stone-600">{{ t('common.loading') }}</p>
    <template v-else>
      <div class="space-y-3">
      <h3 class="text-lg font-semibold">{{ t('coverage.choose_day') }}</h3>
      <ul class="grid grid-cols-2 gap-2 sm:grid-cols-3" :aria-label="t('coverage.days')">
        <li v-for="d in calendarDays" :key="d.date">
          <button
            type="button"
            class="w-full rounded-lg border px-3 py-3 text-start transition cursor-pointer"
            :class="selectedDay === d.date ? 'bg-(--brand) text-(--brand-text) border-(--brand-foreground) font-semibold' : 'border-stone-200 bg-white hover:border-stone-400'"
            :aria-pressed="selectedDay === d.date"
            @click="selectedDay = d.date"
          >
            <span class="text-sm">{{ dayLabel(d.date) }}</span>
            <UIcon v-if="selectedDay === d.date" name="i-lucide-check" class="ms-2 size-4 align-middle" aria-hidden="true" />
          </button>
        </li>
      </ul>
      </div>

      <section v-if="day" class="space-y-3" :aria-label="dayLabel(day.date)">
        <h2 class="text-lg font-semibold" aria-live="polite">{{ t('coverage.choose_times', { date: dayLabel(day.date) }) }}</h2>
        <p class="text-sm text-stone-600">{{ t('coverage.pick_help') }}</p>
        <ul class="flex flex-wrap gap-x-5 gap-y-2 rounded-lg border border-stone-200 bg-white p-3 text-sm" :aria-label="t('coverage.legend')">
          <li class="flex items-center gap-2"><span class="size-4 border border-stone-200 border-s-[3px] border-s-(--brand-foreground) rounded" aria-hidden="true" />{{ t('coverage.needs_prayer') }}</li>
          <li class="flex items-center gap-2"><span class="size-4 border border-stone-200 bg-stone-50 rounded" aria-hidden="true" />{{ t('coverage.covered') }}</li>
          <li class="flex items-center gap-2"><span class="size-4 bg-(--brand) text-(--brand-text) rounded flex items-center justify-center" aria-hidden="true"><UIcon name="i-lucide-check" class="size-3" /></span>{{ t('coverage.selected') }}</li>
        </ul>
        <div v-for="group in timeGroups" :key="group.key" class="space-y-3 rounded-lg border border-stone-200 bg-white p-4">
        <h3 class="font-semibold text-sm">{{ t(`coverage.${group.key}`) }}</h3>
        <div class="grid gap-2" :class="embedded ? 'grid-cols-2 sm:grid-cols-3' : 'grid-cols-2 sm:grid-cols-4 md:grid-cols-6'">
          <button
            v-for="s in group.slots"
            :key="s.i"
            type="button"
            :disabled="s.past"
            :aria-pressed="s.picked"
            class="rounded-lg border px-3 py-3 text-sm flex flex-col items-start gap-1 cursor-pointer disabled:cursor-not-allowed disabled:opacity-40"
            :class="s.picked ? 'bg-(--brand) text-(--brand-text) border-(--brand-foreground)' : s.count === 0 && !s.past ? 'bg-white border-stone-200 border-s-[3px] border-s-(--brand-foreground) hover:border-(--brand-foreground)' : 'bg-stone-50 border-stone-200 hover:border-(--brand-foreground)'"
            @click="toggle(s.iso)"
          >
            <span class="flex w-full items-center justify-between gap-2">
              <span class="font-medium">{{ s.label }}</span>
              <span class="font-semibold tabular-nums" :aria-label="t('coverage.people_in_slot', { n: s.count })">{{ num(s.count) }}</span>
            </span>
            <span v-if="s.picked" class="flex items-center gap-1 text-xs"><UIcon name="i-lucide-check" class="size-3" />{{ t('coverage.selected') }}</span>
            <span v-else-if="s.count === 0 && !s.past" class="text-xs font-semibold text-(--brand-foreground)">{{ t('coverage.needs_prayer') }}</span>
            <span v-else class="text-xs" :aria-label="t('coverage.people_in_slot', { n: s.count })">{{ t('coverage.people_in_slot', { n: s.count }) }}</span>
          </button>
        </div>
        </div>
      </section>

      <div v-if="picks.length && !embedded" class="rounded-lg bg-white border border-stone-300 p-4 space-y-3" :class="!embedded ? 'sticky bottom-4 shadow-lg' : ''">
        <ul class="flex flex-wrap gap-2 max-h-32 overflow-y-auto">
          <li v-for="pick in picks" :key="pick" class="flex items-center gap-2 rounded-lg bg-stone-100 px-3 py-2 text-sm">
            {{ pickLabel(pick) }}
            <button type="button" class="cursor-pointer" :aria-label="`${t('commitment.remove')}: ${pickLabel(pick)}`" @click="toggle(pick)"><UIcon name="i-lucide-x" class="size-4" /></button>
          </li>
        </ul>
        <div class="flex flex-wrap items-center gap-3">
        <span class="font-medium">{{ t('coverage.picked', { n: picks.length }, picks.length) }}</span>
        <UButton v-if="!embedded" :to="signupLink" :label="t('coverage.signup_picked')" trailing-icon="i-lucide-arrow-right" class="campaign-signup-button" />
        <UButton variant="ghost" color="neutral" :label="t('coverage.clear')" @click="picks = []" />
        </div>
      </div>
    </template>
  </div>
</template>

<style scoped>
.campaign-signup-button {
  background-color: var(--brand);
  color: var(--brand-text);
}
.campaign-signup-button:hover,
.campaign-signup-button:active {
  background-color: var(--brand);
  color: var(--brand-text);
}
</style>

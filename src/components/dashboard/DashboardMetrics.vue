<script setup>
import { computed } from 'vue'
const props = defineProps({ overview: { type: Object, required: true } })
const cards = computed(() => [
  { label: '团队成员', value: props.overview.totalUsers, sub: `近期开通 ${props.overview.newUsers} 人`, tone: 'blue' },
  { label: '全部任务', value: props.overview.totalTasks, sub: `待开始 ${props.overview.pending} 项`, tone: 'purple' },
  { label: '进行中', value: props.overview.inProgress, sub: `逾期 ${props.overview.overdue} 项`, tone: 'orange' },
  { label: '已完成', value: props.overview.completed, sub: `完成率 ${props.overview.totalTasks ? Math.round(props.overview.completed / props.overview.totalTasks * 100) : 0}%`, tone: 'green' }
])
</script>
<template><div class="metric-grid"><article v-for="card in cards" :key="card.label" :class="['metric',card.tone]"><span>{{ card.label }}</span><strong>{{ card.value }}</strong><small>{{ card.sub }}</small></article></div></template>
<style scoped>
.metric-grid { display:grid; grid-template-columns:repeat(4,minmax(0,1fr)); gap:18px; }.metric { background:white; border:1px solid #e4ebf5; padding:24px; border-radius:14px; border-top:3px solid #5083eb; }.metric span { display:block; color:#71829a; font-size:13px; }.metric strong { display:block; margin:16px 0; font-size:34px; font-weight:650; letter-spacing:-1px; color:#2f4663; }.metric small { color:#8796a9; font-size:12px; }.purple { border-top-color:#9983e8; }.orange { border-top-color:#e9ac66; }.green { border-top-color:#54bb9c; }@media(max-width:900px) { .metric-grid { grid-template-columns:repeat(2,minmax(0,1fr)); gap:12px; }.metric { padding:18px; } }
</style>

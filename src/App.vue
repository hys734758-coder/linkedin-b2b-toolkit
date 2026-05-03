<template>
  <nav class="nav">
    <div class="nav-inner">
      <span class="nav-logo">
        <svg width="22" height="22" viewBox="0 0 24 24" fill="#0a66c2"><path d="M20.447 20.452h-3.554v-5.569c0-1.328-.027-3.037-1.852-3.037-1.853 0-2.136 1.445-2.136 2.939v5.667H9.351V9h3.414v1.561h.046c.477-.9 1.637-1.85 3.37-1.85 3.601 0 4.267 2.37 4.267 5.455v6.286zM5.337 7.433a2.062 2.062 0 01-2.063-2.065 2.064 2.064 0 112.063 2.065zm1.782 13.019H3.555V9h3.564v11.452zM22.225 0H1.771C.792 0 0 .774 0 1.729v20.542C0 23.227.792 24 1.771 24h20.451C23.2 24 24 23.227 24 22.271V1.729C24 .774 23.2 0 22.222 0h.003z"/></svg>
        LinkedIn B2B 获客工具箱
      </span>
      <div class="nav-tabs">
        <button v-for="tab in tabs" :key="tab.id"
          class="nav-tab" :class="{active: activeTab===tab.id}"
          @click="activeTab=tab.id">{{ tab.label }}</button>
      </div>
    </div>
  </nav>

  <main class="main">
    <CompetitorAnalyzer v-if="activeTab==='competitor'" />
    <OutreachWriter v-else-if="activeTab==='outreach'" />
    <ProfileVisiting v-else-if="activeTab==='visiting'" />
  </main>
</template>

<script setup>
import { ref } from 'vue'
import CompetitorAnalyzer from './components/CompetitorAnalyzer.vue'
import OutreachWriter from './components/OutreachWriter.vue'
import ProfileVisiting from './components/ProfileVisiting.vue'

const tabs = [
  { id: 'competitor', label: '🏢 竞品分析' },
  { id: 'outreach', label: '✉️ 开发信改写' },
  { id: 'visiting', label: '👣 访客足迹挖掘' },
]
const activeTab = ref('competitor')
</script>

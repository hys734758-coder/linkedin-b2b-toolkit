<template>
  <div>
    <h1 class="page-title">竞品/同行 LinkedIn 分析器</h1>
    <p class="page-desc">输入竞品或目标公司的 LinkedIn 主页 URL，通过 Remotive API 的职位数据反推其核心技能关键词需求，对比分析人才策略。</p>

    <div class="card">
      <div class="row">
        <div class="grow">
          <label>公司名称</label>
          <input type="text" v-model="company" placeholder="例：Stripe, Notion, Vercel"
            @keyup.enter="analyze" />
        </div>
        <div>
          <label>职位关键词（可选过滤）</label>
          <input type="text" v-model="roleFilter" placeholder="例：engineer, designer" />
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="analyze" :disabled="loading">
            {{ loading ? '分析中…' : '开始分析' }}
          </button>
        </div>
      </div>
      <div style="margin-top:10px">
        <span v-for="s in sampleCompanies" :key="s" class="suggest-badge" @click="company=s;analyze()">{{ s }}</span>
      </div>
    </div>

    <template v-if="loaded">
      <!-- Company overview -->
      <div class="card">
        <div class="card-title">分析结果：{{ company }}</div>
        <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(110px,1fr));gap:10px">
          <div class="metric-card">
            <div class="metric-label">匹配职位</div>
            <div class="metric-value">{{ matchedJobs.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">不同职位标题</div>
            <div class="metric-value">{{ uniqueTitles }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">技能标签</div>
            <div class="metric-value">{{ allTags.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">有薪资信息</div>
            <div class="metric-value">{{ jobsWithSalary }}</div>
          </div>
        </div>
      </div>

      <!-- Top Skills Radar -->
      <div class="card">
        <div class="card-title">核心技能关键词 TOP 20</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">从职位标签中提取，反映该公司的人才需求重心</p>
        <div v-for="tag in topSkills.slice(0, 20)" :key="tag.name"
          style="display:flex;align-items:center;gap:8px;padding:3px 0">
          <div style="width:100px;font-size:12px;text-align:right;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="tag.name">{{ tag.name }}</div>
          <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
            <div :style="`width:${tag.pct}%;height:100%;border-radius:3px;background:hsl(${160+tag.rank*12},60%,${38+tag.ratio*25}%)`"></div>
          </div>
          <div style="width:28px;font-size:11px;color:var(--gray-600);text-align:right">{{ tag.count }}</div>
        </div>
      </div>

      <!-- Skill Categorization -->
      <div class="card">
        <div class="card-title">技能分类洞察</div>
        <div class="grid-2">
          <div>
            <div class="section-label" style="color:#1e40af">技术 / 工具类</div>
            <div class="tag-list">
              <span v-for="s in techSkills" :key="s" class="kw-chip kw-med" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
          </div>
          <div>
            <div class="section-label" style="color:#92400e">商业 / 软技能类</div>
            <div class="tag-list">
              <span v-for="s in softSkills" :key="s" class="kw-chip kw-high" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Role Distribution -->
      <div class="card" v-if="roleStats.length">
        <div class="card-title">职位类型分布</div>
        <div v-for="role in roleStats" :key="role.title"
          style="display:flex;align-items:center;gap:8px;padding:4px 0">
          <div style="width:180px;font-size:12px;flex-shrink:0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis" :title="role.title">{{ role.title }}</div>
            <div style="flex:1;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
              <div :style="`width:${role.pct}%;height:100%;border-radius:3px;background:var(--blue)`"></div>
            </div>
          <div style="width:28px;font-size:11px;color:var(--gray-600);text-align:right">{{ role.count }}</div>
        </div>
      </div>

      <!-- Job Type -->
      <div class="card">
        <div class="card-title">雇佣类型分布</div>
        <div class="row" style="flex-wrap:wrap;gap:16px">
          <div v-for="jt in jobTypeStats" :key="jt.type" style="display:flex;align-items:center;gap:8px">
            <span class="type-badge">{{ jt.type }}</span>
            <div style="width:80px;height:6px;background:var(--gray-100);border-radius:3px;overflow:hidden">
              <div :style="`width:${jt.pct}%;height:100%;border-radius:3px;background:var(--blue)`"></div>
            </div>
            <span style="font-size:12px;color:var(--gray-600)">{{ jt.count }} ({{ jt.pct }}%)</span>
          </div>
        </div>
      </div>

      <!-- Actionable insights -->
      <div class="card">
        <div class="card-title">策略建议</div>
        <div v-for="(insight, i) in insights" :key="i"
          style="display:flex;gap:10px;padding:8px 0;border-bottom:0.5px solid var(--gray-100)">
          <span style="font-size:14px;flex-shrink:0">{{ insight.icon }}</span>
          <div style="font-size:13px;color:var(--gray-800);line-height:1.6">{{ insight.text }}</div>
        </div>
      </div>

      <!-- Jobs list -->
      <div class="card">
        <div class="card-title">相关职位列表</div>
        <div v-if="matchedJobs.length">
          <div v-for="job in matchedJobs" :key="job.id" class="job-item">
            <div style="display:flex;align-items:center;gap:10px;margin-bottom:4px">
              <img v-if="job.company_logo" :src="job.company_logo"
                style="width:32px;height:32px;border-radius:6px;object-fit:cover"
                loading="lazy" @error="$event.target.style.display='none'" />
              <div style="flex:1;min-width:0">
                <a :href="job.url" target="_blank" rel="noopener"
                  style="font-size:13px;font-weight:500;color:var(--blue);text-decoration:none;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">{{ job.title }}</a>
                <div style="font-size:11px;color:var(--gray-600)">
                  {{ job.salary || '薪资未公开' }} · {{ job.job_type }} · {{ formatDate(job.publication_date) }}
                </div>
              </div>
            </div>
            <div class="tag-list" style="margin:0">
              <span v-for="tag in (job.tags||[]).slice(0,6)" :key="tag" class="kw-chip kw-med" style="font-size:11px;padding:1px 7px">{{ tag }}</span>
            </div>
          </div>
        </div>
        <p v-else style="color:var(--gray-400);font-size:13px">
          没有找到该公司在 Remotive 数据库中的职位。该公司可能不招远程职位，或公司名称需要调整。
        </p>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const API_BASE = 'https://remotive.com/api/remote-jobs'
const company = ref('')
const roleFilter = ref('')
const loading = ref(false)
const loaded = ref(false)
const matchedJobs = ref([])

const sampleCompanies = ['Stripe', 'Notion', 'Vercel', 'Shopify', 'GitLab', 'Automattic', 'Deel', 'Webflow']

const TECH_KEYWORDS = new Set([
  'python','javascript','typescript','react','vue','node','docker','kubernetes','aws','gcp','azure',
  'sql','postgresql','redis','mongodb','graphql','rest api','ci/cd','terraform','linux',
  'figma','sketch','tailwind','html','css','java','go','rust','swift','kotlin','flutter',
  'tensorflow','pytorch','spark','airflow','kafka','elasticsearch','tableau','power bi',
  'c++','ruby','rails','django','flask','spring','angular','next.js','nuxt',
  'machine learning','data engineering','data science','devops','sre',
  'saas','api','microservices','serverless','blockchain','ai/ml','deep learning',
])

async function analyze() {
  if (!company.value.trim() || loading.value) return
  loading.value = true
  const q = company.value.trim()
  try {
    // Remotive doesn't support company search directly, search by company name
    const params = new URLSearchParams({ search: q, limit: '100' })
    if (roleFilter.value.trim()) params.set('search', `${q} ${roleFilter.value.trim()}`)
    const res = await fetch(`${API_BASE}?${params}`)
    const data = await res.json()
    matchedJobs.value = (data.jobs || []).filter(j =>
      j.company_name?.toLowerCase().includes(q.toLowerCase())
    )
    loaded.value = true
  } catch (err) {
    alert('请求失败: ' + err.message)
  } finally {
    loading.value = false
  }
}

const topSkills = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    (j.tags || []).forEach(t => {
      const k = t.toLowerCase()
      freq[k] = (freq[k] || 0) + 1
    })
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([name, count], i) => ({
      name,
      count,
      ratio: count / maxCount,
      pct: Math.round(count / maxCount * 100),
      rank: i,
    }))
    .sort((a, b) => b.count - a.count)
})

const allTags = computed(() => topSkills.value)

const techSkills = computed(() =>
  topSkills.value.filter(t => TECH_KEYWORDS.has(t.name)).map(t => t.name).slice(0, 15)
)
const softSkills = computed(() =>
  topSkills.value.filter(t => !TECH_KEYWORDS.has(t.name)).map(t => t.name).slice(0, 15)
)

const uniqueTitles = computed(() =>
  new Set(matchedJobs.value.map(j => j.title)).size
)

const jobsWithSalary = computed(() =>
  matchedJobs.value.filter(j => j.salary).length
)

const roleStats = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    const t = j.title || 'Unknown'
    freq[t] = (freq[t] || 0) + 1
  })
  const maxCount = Math.max(...Object.values(freq), 1)
  return Object.entries(freq)
    .map(([title, count]) => ({ title, count, pct: Math.round(count / maxCount * 100) }))
    .sort((a, b) => b.count - a.count)
    .slice(0, 8)
})

const jobTypeStats = computed(() => {
  const total = matchedJobs.value.length || 1
  const freq = {}
  matchedJobs.value.forEach(j => {
    const t = (j.job_type || 'other')
    freq[t] = (freq[t] || 0) + 1
  })
  return Object.entries(freq)
    .map(([type, count]) => ({ type, count, pct: Math.round(count / total * 100) }))
    .sort((a, b) => b.count - a.count)
})

const insights = computed(() => {
  const items = []
  if (!matchedJobs.value.length) return items

  const top3 = topSkills.value.slice(0, 3).map(t => t.name)
  if (top3.length) {
    items.push({ icon: '🎯', text: `该公司最核心的技能需求是：${top3.join('、')}。在你的 LinkedIn 资料中突出这些关键词能显著提升匹配度。` })
  }

  const remotePct = Math.round(jobTypeStats.value.reduce((s, j) => j.type === 'full_time' ? s + j.count : s, 0) / (matchedJobs.value.length || 1) * 100)
  items.push({ icon: '🏠', text: `${matchedJobs.value.length} 个远程职位中，全职占 ${remotePct}%，说明该公司有稳定的远程人才招聘策略。` })

  const techCount = techSkills.value.length
  const softCount = softSkills.value.length
  if (techCount > softCount) {
    items.push({ icon: '💻', text: `技术技能标签 (${techCount}) 远多于软技能 (${softCount})，建议简历中量化展示技术项目成果。` })
  } else {
    items.push({ icon: '🤝', text: `软技能标签 (${softCount}) 占比较高，说明该公司重视综合素质，建议在资料中突出跨部门协作和沟通能力。` })
  }

  if (jobsWithSalary.value === 0) {
    items.push({ icon: '💰', text: '所有职位均未公开薪资信息，这通常意味着薪资范围较灵活或有谈判空间。' })
  }

  return items
})

function formatDate(d) {
  if (!d) return ''
  try { return new Date(d).toLocaleDateString('zh-CN') } catch { return d }
}

async function copyWord(w) {
  try { await navigator.clipboard.writeText(w) } catch {}
}
</script>

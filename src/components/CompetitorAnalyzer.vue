<template>
  <div>
    <h1 class="page-title">竞品/同行 LinkedIn 分析器</h1>
    <p class="page-desc">输入竞品或目标公司名称，自动获取公司资料、职位数据、技能画像，全方位分析人才策略与竞争格局。</p>

    <!-- Search -->
    <div class="card">
      <div class="row">
        <div class="grow">
          <label>公司名称</label>
          <input type="text" v-model="company" placeholder="例：Stripe, Notion, Vercel"
            @keyup.enter="analyze" />
        </div>
        <div style="padding-top:22px">
          <button class="btn btn-primary" @click="analyze" :disabled="loading">
            {{ loading ? '分析中…' : '开始分析' }}
          </button>
        </div>
      </div>
      <div style="margin-top:10px">
        <span v-for="s in sampleCompanies" :key="s" class="suggest-badge"
          @click="company=s;analyze()">{{ s }}</span>
      </div>
      <div v-if="apiStatus.length" style="margin-top:12px;display:flex;flex-wrap:wrap;gap:6px">
        <span v-for="s in apiStatus" :key="s.name" class="api-badge"
          :class="s.ok?'api-ok':'api-err'">
          {{ s.ok ? '✓' : '✗' }} {{ s.name }}
        </span>
      </div>
    </div>

    <template v-if="loaded">
      <!-- Company Profile Card -->
      <div class="card" v-if="companyProfile">
        <div class="card-title">公司概览</div>
        <div style="display:flex;gap:20px;align-items:flex-start;flex-wrap:wrap">
          <div v-if="companyProfile.logo" style="flex-shrink:0">
            <img :src="companyProfile.logo" :alt="companyProfile.name"
              style="width:80px;height:80px;border-radius:12px;object-fit:cover;border:1px solid var(--gray-200)"
              @error="$event.target.style.display='none'" />
          </div>
          <div style="flex:1;min-width:200px">
            <div style="font-size:18px;font-weight:600;color:var(--gray-900)">{{ companyProfile.name }}</div>
            <div style="margin-top:6px;display:flex;flex-wrap:wrap;gap:8px">
              <a v-if="companyProfile.domain" :href="'https://'+companyProfile.domain"
                target="_blank" rel="noopener" class="ext-link">
                🌐 {{ companyProfile.domain }}
              </a>
              <a v-if="companyProfile.linkedin"
                :href="companyProfile.linkedin" target="_blank" rel="noopener" class="ext-link ext-link-blue">
                💼 LinkedIn 主页
              </a>
            </div>
            <div v-if="companyProfile.description" style="margin-top:8px;font-size:13px;color:var(--gray-600);line-height:1.6">
              {{ companyProfile.description }}
            </div>
          </div>
        </div>
      </div>

      <!-- Key Metrics -->
      <div class="card">
        <div class="card-title">数据总览</div>
        <div style="display:grid;grid-template-columns:repeat(auto-fit,minmax(100px,1fr));gap:10px">
          <div class="metric-card">
            <div class="metric-label">匹配职位</div>
            <div class="metric-value">{{ matchedJobs.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">职位标题数</div>
            <div class="metric-value">{{ uniqueTitles }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">技能标签</div>
            <div class="metric-value">{{ topSkills.length }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">有薪资信息</div>
            <div class="metric-value">{{ jobsWithSalary }}</div>
          </div>
          <div class="metric-card">
            <div class="metric-label">数据来源</div>
            <div class="metric-value" style="font-size:12px">{{ dataSource }}</div>
          </div>
        </div>
      </div>

      <!-- Top Skills -->
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
        <div v-if="!topSkills.length" style="color:var(--gray-400);font-size:13px;padding:12px 0">
          暂无技能标签数据
        </div>
      </div>

      <!-- Skill Categorization -->
      <div class="card" v-if="techSkills.length || softSkills.length">
        <div class="card-title">技能分类洞察</div>
        <div class="grid-2">
          <div>
            <div class="section-label" style="color:#1e40af">技术 / 工具类 ({{ techSkills.length }})</div>
            <div class="tag-list" v-if="techSkills.length">
              <span v-for="s in techSkills" :key="s" class="kw-chip kw-med" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
            <div v-else style="font-size:12px;color:var(--gray-400)">未检测到</div>
          </div>
          <div>
            <div class="section-label" style="color:#92400e">商业 / 软技能类 ({{ softSkills.length }})</div>
            <div class="tag-list" v-if="softSkills.length">
              <span v-for="s in softSkills" :key="s" class="kw-chip kw-high" style="cursor:pointer" @click="copyWord(s)">{{ s }}</span>
            </div>
            <div v-else style="font-size:12px;color:var(--gray-400)">未检测到</div>
          </div>
        </div>
      </div>

      <!-- Role Distribution -->
      <div class="card" v-if="roleStats.length">
        <div class="card-title">职位类型分布 TOP 10</div>
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
      <div class="card" v-if="jobTypeStats.length">
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

      <!-- Category Distribution -->
      <div class="card" v-if="categoryStats.length">
        <div class="card-title">职能类别分布</div>
        <div style="display:flex;flex-wrap:wrap;gap:8px">
          <div v-for="cat in categoryStats" :key="cat.name" class="cat-chip">
            <span style="font-weight:500">{{ cat.name }}</span>
            <span style="color:var(--gray-500)">{{ cat.count }}</span>
          </div>
        </div>
      </div>

      <!-- Salary Insights -->
      <div class="card" v-if="salaryInsights.length">
        <div class="card-title">薪资洞察</div>
        <div v-for="si in salaryInsights" :key="si.title" class="salary-item">
          <div style="font-size:13px;font-weight:500;color:var(--gray-800)">{{ si.title }}</div>
          <div style="font-size:12px;color:var(--gray-600)">{{ si.salary }}</div>
        </div>
      </div>

      <!-- Company Suggestions from Clearbit -->
      <div class="card" v-if="suggestedCompanies.length">
        <div class="card-title">相关公司推荐</div>
        <p style="font-size:12px;color:var(--gray-600);margin-bottom:12px">基于名称相似度推荐，点击可切换分析</p>
        <div style="display:flex;flex-wrap:wrap;gap:8px">
          <div v-for="c in suggestedCompanies" :key="c.domain" class="company-chip"
            @click="company=c.name;analyze()" :title="'域名: '+c.domain">
            <img v-if="c.logo" :src="c.logo" style="width:20px;height:20px;border-radius:4px" @error="$event.target.style.display='none'" />
            <span>{{ c.name }}</span>
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
        <div class="card-title">相关职位列表 ({{ matchedJobs.length }})</div>
        <div v-if="matchedJobs.length">
          <div style="margin-bottom:10px">
            <input type="text" v-model="jobSearch" placeholder="搜索职位标题..."
              style="width:100%;max-width:300px;padding:6px 10px;border:1px solid var(--gray-200);border-radius:6px;font-size:12px" />
          </div>
          <div v-for="job in filteredJobs" :key="job.id" class="job-item">
            <div style="display:flex;align-items:center;gap:10px;margin-bottom:4px">
              <img v-if="job.company_logo || job.company_logo_url" :src="job.company_logo || job.company_logo_url"
                style="width:32px;height:32px;border-radius:6px;object-fit:cover"
                loading="lazy" @error="$event.target.style.display='none'" />
              <div style="flex:1;min-width:0">
                <a :href="job.url" target="_blank" rel="noopener"
                  style="font-size:13px;font-weight:500;color:var(--blue);text-decoration:none;display:block;white-space:nowrap;overflow:hidden;text-overflow:ellipsis">{{ job.title }}</a>
                <div style="font-size:11px;color:var(--gray-600)">
                  {{ job.company_name || '' }} · {{ job.salary || '薪资未公开' }} · {{ formatJobType(job.job_type) }} · {{ formatDate(job.publication_date) }}
                </div>
                <div v-if="job.candidate_required_location" style="font-size:11px;color:var(--gray-500)">
                  📍 {{ job.candidate_required_location }}
                </div>
              </div>
              <div v-if="job.category" class="cat-badge">{{ job.category }}</div>
            </div>
            <div class="tag-list" style="margin:0">
              <span v-for="tag in (job.tags||[]).slice(0,8)" :key="tag" class="kw-chip kw-med" style="font-size:11px;padding:1px 7px">{{ tag }}</span>
            </div>
          </div>
        </div>
        <p v-else style="color:var(--gray-400);font-size:13px">
          没有找到匹配的职位数据。可能原因：该公司不招远程职位，或公司名称需要调整。建议尝试使用上方推荐的公司名称。
        </p>
      </div>

      <!-- Export -->
      <div class="card" style="text-align:center">
        <button class="btn" @click="exportReport" :disabled="!matchedJobs.length">📄 导出分析报告</button>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

// ─── State ───
const company = ref('')
const loading = ref(false)
const loaded = ref(false)
const matchedJobs = ref([])
const companyProfile = ref(null)
const suggestedCompanies = ref([])
const apiStatus = ref([])
const jobSearch = ref('')

const sampleCompanies = ['Stripe', 'Notion', 'Vercel', 'Shopify', 'GitLab', 'Deel', 'Webflow', 'OpenAI']

// ─── Tech keyword database (expanded) ───
const TECH_KEYWORDS = new Set([
  'python','javascript','typescript','react','vue','node','node.js','docker','kubernetes','aws','gcp','azure',
  'sql','postgresql','redis','mongodb','graphql','rest api','ci/cd','terraform','linux',
  'figma','sketch','tailwind','html','css','java','go','golang','rust','swift','kotlin','flutter',
  'tensorflow','pytorch','spark','airflow','kafka','elasticsearch','tableau','power bi',
  'c++','c#','ruby','rails','django','flask','spring','angular','next.js','nuxt',
  'machine learning','data engineering','data science','devops','sre',
  'saas','api','microservices','serverless','blockchain','ai/ml','deep learning',
  '.net','unity','unreal','laravel','symfony','svelte','solidity',
  'react native','data analysis','automation','rust','ruby/rails',
  'gcp','data engineering','shopify','video','salesforce','hubspot',
  'snowflake','databricks','bigquery','redshift','neo4j','cassandra',
  'jenkins','github actions','argocd','ansible','puppet','chef',
  'webpack','vite','rollup','babel','eslint','jest','cypress','playwright',
  'react.js','vue.js','express','nestjs','fastapi','gin',
])

// ─── API Endpoints ───
const CLEARBIT_SUGGEST = 'https://autocomplete.clearbit.com/v1/companies/suggest'
const REMOTIVE_API = 'https://remotive.com/api/remote-jobs'

// ─── Main Analysis ───
async function analyze() {
  if (!company.value.trim() || loading.value) return
  loading.value = true
  loaded.value = false
  matchedJobs.value = []
  companyProfile.value = null
  suggestedCompanies.value = []
  apiStatus.value = []

  const q = company.value.trim()

  try {
    // Fire all requests in parallel
    const [clearbitResult, remotiveResult] = await Promise.allSettled([
      fetchCompanyProfile(q),
      fetchJobsFromRemotive(q),
    ])

    // Process Clearbit
    if (clearbitResult.status === 'fulfilled' && clearbitResult.value) {
      companyProfile.value = clearbitResult.value.profile
      suggestedCompanies.value = clearbitResult.value.suggestions
      apiStatus.value.push({ name: 'Clearbit 公司信息', ok: true })
    } else {
      apiStatus.value.push({ name: 'Clearbit 公司信息', ok: false })
    }

    // Process Remotive
    if (remotiveResult.status === 'fulfilled') {
      matchedJobs.value = remotiveResult.value
      apiStatus.value.push({ name: 'Remotive 远程职位', ok: true })
    } else {
      apiStatus.value.push({ name: 'Remotive 远程职位', ok: false })
    }

    loaded.value = true
  } catch (err) {
    console.error('Analysis error:', err)
    alert('分析过程中出错: ' + err.message)
  } finally {
    loading.value = false
  }
}

// ─── Clearbit: Company profile + suggestions ───
async function fetchCompanyProfile(name) {
  const res = await fetch(`${CLEARBIT_SUGGEST}?query=${encodeURIComponent(name)}`)
  if (!res.ok) throw new Error(`Clearbit API ${res.status}`)
  const data = await res.json()

  if (!data || !data.length) return null

  // First result is the best match
  const best = data[0]
  const profile = {
    name: best.name,
    domain: best.domain,
    logo: best.logo || `https://logo.clearbit.com/${best.domain}`,
    linkedin: best.linkedin
      ? (best.linkedin.startsWith('http') ? best.linkedin : `https://www.linkedin.com${best.linkedin}`)
      : `https://www.linkedin.com/company/${best.domain.split('.')[0]}/`,
    description: best.description || `官网: ${best.domain}`,
  }

  // Remaining suggestions
  const suggestions = data.slice(1, 8).filter(Boolean)

  return { profile, suggestions }
}

// ─── Remotive: Remote jobs ───
async function fetchJobsFromRemotive(name) {
  const q = name.toLowerCase()

  // Try multiple search strategies
  const strategies = [
    { search: name, limit: 100 },
    { search: `${name} remote`, limit: 100 },
  ]

  let allJobs = []
  const seen = new Set()

  for (const params of strategies) {
    try {
      const url = new URL(REMOTIVE_API)
      Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v))

      const res = await fetch(url.toString())
      if (!res.ok) continue
      const data = await res.json()
      const jobs = (data.jobs || []).filter(j => {
        const match = j.company_name?.toLowerCase().includes(q)
          || j.title?.toLowerCase().includes(q)
        return match
      })

      for (const job of jobs) {
        if (!seen.has(job.id)) {
          seen.add(job.id)
          allJobs.push(job)
        }
      }
    } catch {
      continue
    }
  }

  return allJobs
}

// ─── Computed: Skills ───
const topSkills = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    (j.tags || []).forEach(t => {
      const k = t.trim()
      if (!k) return
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

const techSkills = computed(() =>
  topSkills.value.filter(t => TECH_KEYWORDS.has(t.name.toLowerCase())).map(t => t.name).slice(0, 20)
)
const softSkills = computed(() =>
  topSkills.value.filter(t => !TECH_KEYWORDS.has(t.name.toLowerCase())).map(t => t.name).slice(0, 20)
)

const uniqueTitles = computed(() =>
  new Set(matchedJobs.value.map(j => j.title)).size
)

const jobsWithSalary = computed(() =>
  matchedJobs.value.filter(j => j.salary).length
)

const dataSource = computed(() => {
  const sources = []
  if (matchedJobs.value.length) sources.push('Remotive')
  if (companyProfile.value) sources.push('Clearbit')
  return sources.join(' + ') || '无'
})

// ─── Computed: Stats ───
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
    .slice(0, 10)
})

const jobTypeStats = computed(() => {
  const total = matchedJobs.value.length || 1
  const freq = {}
  matchedJobs.value.forEach(j => {
    const t = j.job_type || 'other'
    freq[t] = (freq[t] || 0) + 1
  })
  return Object.entries(freq)
    .map(([type, count]) => ({ type: formatJobType(type), count, pct: Math.round(count / total * 100) }))
    .sort((a, b) => b.count - a.count)
})

const categoryStats = computed(() => {
  const freq = {}
  matchedJobs.value.forEach(j => {
    const cat = j.category || 'Other'
    freq[cat] = (freq[cat] || 0) + 1
  })
  return Object.entries(freq)
    .map(([name, count]) => ({ name, count }))
    .sort((a, b) => b.count - a.count)
})

const salaryInsights = computed(() => {
  return matchedJobs.value
    .filter(j => j.salary)
    .slice(0, 8)
    .map(j => ({ title: j.title, salary: j.salary }))
})

const filteredJobs = computed(() => {
  if (!jobSearch.value.trim()) return matchedJobs.value
  const q = jobSearch.value.toLowerCase()
  return matchedJobs.value.filter(j =>
    j.title?.toLowerCase().includes(q) ||
    j.company_name?.toLowerCase().includes(q) ||
    (j.tags || []).some(t => t.toLowerCase().includes(q))
  )
})

// ─── Insights engine ───
const insights = computed(() => {
  const items = []
  if (!matchedJobs.value.length && !companyProfile.value) return items

  const top3 = topSkills.value.slice(0, 3).map(t => t.name)
  if (top3.length) {
    items.push({
      icon: '🎯',
      text: `该公司最核心的技能需求是：${top3.join('、')}。在你的 LinkedIn 资料中突出这些关键词能显著提升匹配度。`
    })
  }

  const fullTime = jobTypeStats.value.find(j => j.type === '全职' || j.type === 'Full-time')
  if (fullTime) {
    items.push({
      icon: '🏠',
      text: `在 ${matchedJobs.value.length} 个远程职位中，全职占 ${fullTime.pct}%，说明该公司有稳定的远程人才招聘策略。`
    })
  }

  const techCount = techSkills.value.length
  const softCount = softSkills.value.length
  if (techCount > softCount) {
    items.push({
      icon: '💻',
      text: `技术技能标签 (${techCount}) 远多于软技能 (${softCount})，建议简历中量化展示技术项目成果。`
    })
  } else if (softCount > 0) {
    items.push({
      icon: '🤝',
      text: `软技能标签 (${softCount}) 占比较高，说明该公司重视综合素质，建议在资料中突出跨部门协作和沟通能力。`
    })
  }

  if (jobsWithSalary.value === 0 && matchedJobs.value.length > 0) {
    items.push({
      icon: '💰',
      text: '所有职位均未公开薪资信息，这通常意味着薪资范围较灵活或有谈判空间。'
    })
  } else if (jobsWithSalary.value > 0) {
    items.push({
      icon: '💰',
      text: `${jobsWithSalary.value} 个职位公开了薪资信息，建议参考这些范围来优化你的薪资期望表述。`
    })
  }

  if (categoryStats.value.length > 3) {
    const topCats = categoryStats.value.slice(0, 3).map(c => `${c.name}(${c.count})`).join('、')
    items.push({
      icon: '📊',
      text: `主要招聘类别：${topCats}，说明该公司正在这些方向扩大团队。`
    })
  }

  if (companyProfile.value?.domain) {
    items.push({
      icon: '🔗',
      text: `该公司官网为 ${companyProfile.value.domain}，建议先了解其产品和服务再进行接触，有针对性的开发信效果更好。`
    })
  }

  if (suggestedCompanies.value.length > 0) {
    items.push({
      icon: '🔄',
      text: `还发现了 ${suggestedCompanies.value.length} 个相关公司（如 ${suggestedCompanies.value.slice(0, 3).map(c => c.name).join('、')}），可以逐一分析，扩展竞品地图。`
    })
  }

  if (matchedJobs.value.length === 0 && companyProfile.value) {
    items.push({
      icon: '🔍',
      text: '未找到该公司在 Remotive 远程职位数据库中的记录。该公司可能不招远程职位，但仍然是有价值的潜在客户。建议访问其 LinkedIn 主页了解公司动态。'
    })
  }

  return items
})

// ─── Export ───
function exportReport() {
  const lines = [
    `# 竞品分析报告: ${company.value}`,
    `生成时间: ${new Date().toLocaleString('zh-CN')}`,
    '',
    '## 公司概览',
    companyProfile.value
      ? `- 名称: ${companyProfile.value.name}\n- 域名: ${companyProfile.value.domain}\n- LinkedIn: ${companyProfile.value.linkedin}`
      : '- 未找到公司信息',
    '',
    '## 数据总览',
    `- 匹配职位: ${matchedJobs.value.length}`,
    `- 职位标题数: ${uniqueTitles.value}`,
    `- 技能标签数: ${topSkills.value.length}`,
    '',
    '## TOP 10 技能关键词',
    ...topSkills.value.slice(0, 10).map((t, i) => `${i + 1}. ${t.name} (${t.count})`),
    '',
    '## 策略建议',
    ...insights.value.map(ins => `- ${ins.text}`),
    '',
    '## 职位列表',
    ...matchedJobs.value.map(j =>
      `- [${j.title}](${j.url}) | ${j.company_name || ''} | ${j.salary || 'N/A'} | ${j.job_type || 'N/A'}`
    ),
  ]

  const blob = new Blob([lines.join('\n')], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `竞品分析_${company.value}_${new Date().toISOString().slice(0, 10)}.md`
  a.click()
  URL.revokeObjectURL(url)
}

// ─── Helpers ───
function formatDate(d) {
  if (!d) return ''
  try { return new Date(d).toLocaleDateString('zh-CN') } catch { return d }
}

function formatJobType(t) {
  if (!t) return '未知'
  const map = {
    full_time: '全职',
    part_time: '兼职',
    contract: '合同工',
    freelance: '自由职业',
    internship: '实习',
  }
  return map[t.toLowerCase()] || t
}

async function copyWord(w) {
  try { await navigator.clipboard.writeText(w) } catch {}
}
</script>

<style scoped>
.ext-link {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--gray-600);
  text-decoration: none;
  padding: 3px 8px;
  border: 1px solid var(--gray-200);
  border-radius: 6px;
  transition: all 0.15s;
}
.ext-link:hover {
  background: var(--gray-50);
  border-color: var(--gray-300);
}
.ext-link-blue:hover {
  border-color: #0a66c2;
  color: #0a66c2;
}

.api-badge {
  display: inline-flex;
  align-items: center;
  gap: 3px;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
}
.api-ok {
  background: #f0fdf4;
  color: #166534;
  border: 1px solid #bbf7d0;
}
.api-err {
  background: #fef2f2;
  color: #991b1b;
  border: 1px solid #fecaca;
}

.company-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  border: 1px solid var(--gray-200);
  border-radius: 8px;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.15s;
}
.company-chip:hover {
  background: var(--gray-50);
  border-color: var(--blue);
  color: var(--blue);
}

.cat-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 10px;
  background: var(--gray-50);
  border: 1px solid var(--gray-200);
  border-radius: 6px;
  font-size: 12px;
  color: var(--gray-700);
}
.cat-badge {
  font-size: 11px;
  padding: 2px 6px;
  background: #eff6ff;
  color: #1e40af;
  border-radius: 4px;
  flex-shrink: 0;
}

.salary-item {
  padding: 6px 0;
  border-bottom: 0.5px solid var(--gray-100);
}

.job-item {
  padding: 10px 0;
  border-bottom: 0.5px solid var(--gray-100);
}
.job-item:last-child {
  border-bottom: none;
}
</style>

<template>
  <div class="outreach-writer">
    <h2>✉️ LinkedIn 开发信 AI 改写器</h2>
    <p class="subtitle">输入中文需求，AI 自动生成高转化率英文开发信，附带垃圾邮件风险检测</p>

    <!-- 免费 API 快速设置 -->
    <div class="api-section">
      <div class="api-tabs">
        <button class="api-tab" :class="{active: apiMode==='free'}" @click="apiMode='free'">🆓 免费 Gemini API（推荐）</button>
        <button class="api-tab" :class="{active: apiMode==='custom'}" @click="apiMode='custom'">⚙️ 自定义 API</button>
        <button class="api-tab" :class="{active: apiMode==='mock'}" @click="apiMode='mock'">📋 模拟模式</button>
      </div>

      <!-- 免费模式 -->
      <div v-if="apiMode==='free'" class="api-panel free-panel">
        <div class="free-steps">
          <div class="step">
            <span class="step-num">1</span>
            <div class="step-content">
              <strong>获取免费 Gemini API Key</strong>
              <p>访问 <a href="https://aistudio.google.com/apikey" target="_blank">Google AI Studio</a>，登录 Google 账号，点击「Create API key」，无需信用卡，完全免费。</p>
            </div>
          </div>
          <div class="step">
            <span class="step-num">2</span>
            <div class="step-content">
              <strong>粘贴 API Key</strong>
              <div class="key-input-row">
                <input v-model="apiKey" type="password" placeholder="粘贴你的 Gemini API Key（AIza...）" />
                <button class="btn-test" @click="testAPI">测试连接</button>
              </div>
            </div>
          </div>
          <div class="step">
            <span class="step-num">3</span>
            <div class="step-content">
              <strong>开始使用</strong>
              <p>测试通过后，直接在下方面板输入需求，点击「生成开发信」即可。</p>
            </div>
          </div>
        </div>
        <p v-if="apiStatus" :class="apiStatus.type" class="api-msg">{{ apiStatus.msg }}</p>
        <div class="free-info">
          💡 Gemini 免费额度：每分钟 15 次请求，每天 1500 次，完全够用。
        </div>
      </div>

      <!-- 自定义模式 -->
      <div v-if="apiMode==='custom'" class="api-panel">
        <div class="api-row">
          <input v-model="apiEndpoint" placeholder="API Endpoint（OpenAI 兼容格式）" />
          <input v-model="apiKey" type="password" placeholder="API Key（可选）" />
          <button class="btn-link" @click="testAPI">测试连接</button>
        </div>
        <p v-if="apiStatus" :class="apiStatus.type" class="api-msg">{{ apiStatus.msg }}</p>
      </div>

      <!-- 模拟模式 -->
      <div v-if="apiMode==='mock'" class="api-panel mock-panel">
        <p class="mock-info">⚠️ 当前为模拟模式，生成的是内置英文模板，仅供参考。请使用「免费 Gemini API」获取真实 AI 生成效果。</p>
      </div>
    </div>

    <!-- 输入区域 -->
    <div class="input-section">
      <label>中文开发信需求描述：</label>
      <textarea v-model="chineseInput" placeholder="例：我是做机械出口的，想联系美国的大型分销商，介绍我们的产品和价格优势，希望他们回复..." rows="5"></textarea>

      <div class="options-row">
        <div class="option-group">
          <label>目标行业：</label>
          <input v-model="industry" placeholder="如：机械设备、电子、纺织" />
        </div>
        <div class="option-group">
          <label>目标角色：</label>
          <input v-model="targetRole" placeholder="如：采购经理、CEO、业务拓展" />
        </div>
        <div class="option-group">
          <label>邮件风格偏好：</label>
          <select v-model="stylePref">
            <option value="balanced">均衡（3种风格）</option>
            <option value="direct">直接高效型为主</option>
            <option value="friendly">友好亲和型为主</option>
            <option value="value">价值驱动型为主</option>
          </select>
        </div>
      </div>

      <button class="btn-generate" @click="generateMessages" :disabled="!chineseInput.trim() || generating">
        {{ generating ? 'AI 生成中...' : '🚀 生成开发信' }}
      </button>
    </div>

    <!-- 结果区域 -->
    <div v-if="messages.length" class="results-section">
      <h3>📧 生成的开发信（共 {{ messages.length }} 种风格）</h3>

      <div v-for="(msg, idx) in messages" :key="idx" class="message-card" :class="msg.riskLevel">
        <div class="card-header">
          <span class="style-badge" :class="msg.style">{{ styleLabel(msg.style) }}</span>
          <span class="risk-badge" :class="msg.riskLevel">
            {{ riskLabel(msg.riskLevel) }} · 垃圾邮件风险：{{ msg.spamScore }}%
          </span>
        </div>

        <div class="message-content">
          <div class="field">
            <label>主题行：</label>
            <div class="text-block">{{ msg.subject }}</div>
          </div>
          <div class="field">
            <label>正文：</label>
            <div class="text-block" v-html="formatBody(msg.body)"></div>
          </div>
        </div>

        <div class="card-actions">
          <button class="btn-action" @click="copyMessage(msg)">📋 复制</button>
          <button class="btn-action" @click="regenerateOne(idx)">🔄 重新生成此风格</button>
          <div class="spam-analysis">
            <strong>垃圾邮件检测：</strong>
            <span :class="msg.riskLevel">{{ msg.spamAnalysis }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 历史记录 -->
    <div v-if="history.length" class="history-section">
      <h3>📜 历史记录</h3>
      <div v-for="(h, idx) in history" :key="idx" class="history-item" @click="loadHistory(h)">
        <span class="history-preview">{{ h.input.slice(0, 50) }}...</span>
        <span class="history-time">{{ h.time }}</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue'

const chineseInput = ref('')
const industry = ref('')
const targetRole = ref('')
const stylePref = ref('balanced')
const apiEndpoint = ref('')
const apiKey = ref('')
const apiStatus = ref(null)
const generating = ref(false)
const messages = ref([])
const history = ref([])
const apiMode = ref('free') // 'free', 'custom', 'mock'

// 从 localStorage 恢复 API Key 和模式
onMounted(() => {
  const savedKey = localStorage.getItem('lk_toolkit_gemini_key')
  if (savedKey) {
    apiKey.value = savedKey
    apiMode.value = 'free'
  }
  const savedEndpoint = localStorage.getItem('lk_toolkit_api_endpoint')
  if (savedEndpoint) {
    apiEndpoint.value = savedEndpoint
  }
  const savedMode = localStorage.getItem('lk_toolkit_api_mode')
  if (savedMode) {
    apiMode.value = savedMode
  }
})

// 自动保存 API Key 到 localStorage
watch(apiKey, (val) => {
  if (val) {
    localStorage.setItem('lk_toolkit_gemini_key', val)
  } else {
    localStorage.removeItem('lk_toolkit_gemini_key')
  }
})

watch(apiEndpoint, (val) => {
  if (val) localStorage.setItem('lk_toolkit_api_endpoint', val)
  else localStorage.removeItem('lk_toolkit_api_endpoint')
})

watch(apiMode, (val) => {
  localStorage.setItem('lk_toolkit_api_mode', val)
})

const GEMINI_ENDPOINT = 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent'

const SPAM_KEYWORDS = [
  'free', 'guaranteed', 'no obligation', 'act now', 'limited time',
  'cash', 'money', 'income', 'earn extra', 'work from home',
  'buy now', 'order now', 'click here', 'urgent', 'important',
  'congratulations', 'winner', 'prize', '!!!',
  'best price', 'cheapest', 'discount', 'promotion', 'offer',
  '100%', 'no risk', 'risk free', 'special offer', 'limited offer'
]

const styleLabel = (style) => {
  const map = { direct: '🎯 直接高效型', friendly: '🤝 友好亲和型', value: '💡 价值驱动型' }
  return map[style] || style
}

const riskLabel = (level) => {
  const map = { low: '✅ 低风险', medium: '⚠️ 中风险', high: '🚨 高风险' }
  return map[level] || level
}

const formatBody = (body) => {
  return body.replace(/\n/g, '<br>').replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
}

// 垃圾邮件检测
const analyzeSpam = (text) => {
  const lower = text.toLowerCase()
  let score = 0
  const flags = []

  SPAM_KEYWORDS.forEach(word => {
    if (lower.includes(word)) {
      score += 8
      flags.push(`含垃圾邮件词汇: "${word}"`)
    }
  })

  const capsRatio = (text.match(/[A-Z]/g) || []).length / Math.max(text.length, 1)
  if (capsRatio > 0.3) {
    score += 15
    flags.push('过多大写字母')
  }

  const exclCount = (text.match(/!/g) || []).length
  if (exclCount > 2) {
    score += exclCount * 3
    flags.push(`过多感叹号 (${exclCount}个)`)
  }

  if (text.length > 800) {
    score += 10
    flags.push('正文过长（>800字符）')
  } else if (text.length < 100) {
    score += 5
    flags.push('正文过短（<100字符）')
  }

  if (text.includes('http') || text.includes('www.')) {
    score += 20
    flags.push('包含链接（开发信中建议避免）')
  }

  score = Math.min(score, 100)

  let riskLevel, analysis
  if (score <= 20) {
    riskLevel = 'low'
    analysis = `✅ 低风险（${score}分）：此邮件不太可能被标记为垃圾邮件。保持简洁、个性化。`
  } else if (score <= 50) {
    riskLevel = 'medium'
    analysis = `⚠️ 中风险（${score}分）：建议修改：${flags.join('; ')}`
  } else {
    riskLevel = 'high'
    analysis = `🚨 高风险（${score}分）：极易被标记为垃圾邮件！请修改：${flags.join('; ')}`
  }

  return { score, riskLevel, analysis, flags }
}

// 调用 AI API（支持 Gemini 和 OpenAI 格式）
const callAI = async (prompt) => {
  // 模拟模式
  if (apiMode.value === 'mock') {
    return simulateAI(prompt)
  }

  // 免费 Gemini 模式
  if (apiMode.value === 'free') {
    if (!apiKey.value) throw new Error('请先填入 Gemini API Key（免费获取）')
    return callGemini(prompt)
  }

  // 自定义模式
  if (!apiEndpoint.value) return simulateAI(prompt)
  return callOpenAI(prompt)
}

// Gemini API 调用
const callGemini = async (prompt) => {
  const url = `${GEMINI_ENDPOINT}?key=${apiKey.value}`
  const body = {
    contents: [{ parts: [{ text: prompt }] }],
    generationConfig: { temperature: 0.8, maxOutputTokens: 1024 }
  }

  const res = await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  })

  if (!res.ok) {
    const err = await res.json()
    throw new Error(`Gemini API 错误 ${res.status}: ${JSON.stringify(err)}`)
  }

  const data = await res.json()
  if (!data.candidates?.[0]?.content?.parts?.[0]?.text) {
    throw new Error('Gemini 返回格式异常：' + JSON.stringify(data))
  }
  return data.candidates[0].content.parts[0].text
}

// OpenAI 兼容格式调用
const callOpenAI = async (prompt) => {
  const body = {
    model: 'gpt-3.5-turbo',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.8,
  }

  const headers = { 'Content-Type': 'application/json' }
  if (apiKey.value) {
    headers['Authorization'] = `Bearer ${apiKey.value}`
  }

  const res = await fetch(apiEndpoint.value, {
    method: 'POST',
    headers,
    body: JSON.stringify(body),
  })

  if (!res.ok) throw new Error(`API 错误 ${res.status}: ${await res.text()}`)
  const data = await res.json()
  return data.choices[0].message.content
}

// 模拟 AI 生成
const simulateAI = async (prompt) => {
  await new Promise(r => setTimeout(r, 1200))
  return `[模拟模式]
⚠️ 当前为模拟模式，请使用「免费 Gemini API」获取真实 AI 生成效果。

获取免费 Key：https://aistudio.google.com/apikey
1. 访问上述链接，登录 Google 账号
2. 点击「Create API key」
3. 复制 Key 并粘贴到本页面

完全免费，无需信用卡！`
}

// 生成单条消息
const generateSingle = async (style, input, industryVal, roleVal) => {
  const stylePrompt = {
    direct: '直接高效型：开门见山，快速说明价值，尊重对方时间，邮件长度控制在100词以内。使用英语。',
    friendly: '友好亲和型：以共同话题或赞美开头，建立人际关系，自然过渡到商业话题，语气温暖。使用英语。',
    value: '价值驱动型：以对方痛点或行业趋势切入，展示专业知识，提供明确价值主张，引导回复。使用英语。'
  }

  const prompt = `You are a professional LinkedIn外贸开发信 expert. Generate a high-conversion English LinkedIn outreach message.

Target style: ${stylePrompt[style]}

Chinese requirement: ${input}
Target industry: ${industryVal || 'General'}
Target role: ${roleVal || 'Business decision maker'}

Output format (output ONLY the following, no extra text):
SUBJECT: [subject line, max 60 chars, avoid spam words]
BODY:
[body text, 3-5 sentences, personalized, with clear CTA]

Requirements:
1. No spam words (free, guarantee, !!!, etc.)
2. Personalized, mention their company or industry
3. Clear but soft CTA
4. Max 150 words total`

  const output = await callAI(prompt)

  const subjectMatch = output.match(/SUBJECT:\s*(.+)/i)
  const bodyMatch = output.match(/BODY:\s*([\s\S]+)/i)

  let subject = subjectMatch ? subjectMatch[1].trim() : `Partnership Opportunity - ${industryVal || 'Business'}`
  let body = bodyMatch ? bodyMatch[1].trim() : output.replace(/SUBJECT:.*\n?/i, '').trim()

  // 模拟模式使用内置模板
  if (apiMode.value === 'mock' || (apiMode.value === 'free' && !apiKey.value)) {
    const templates = getTemplates(style, input, industryVal, roleVal)
    subject = templates.subject
    body = templates.body
  }

  const fullText = subject + ' ' + body
  const spam = analyzeSpam(fullText)

  return { style, subject, body, ...spam }
}

// 内置模板（模拟模式使用）
const getTemplates = (style, input, industry, role) => {
  const industryPhrase = industry ? ` in the ${industry} industry` : ''
  const rolePhrase = role || 'decision-maker'

  const templates = {
    direct: {
      subject: `Quick question about ${industry || 'your business'}`,
      body: `Hi [Name],

I came across ${rolePhrase === 'CEO' ? 'your company' : 'your profile'}${industryPhrase} and was impressed by your recent growth.

We've helped companies like [Similar Company] reduce costs by 20-30% while maintaining quality. I'd love to share how we might do the same for you.

Would you be open to a brief 10-minute chat this week?

Best,
[Your Name]`
    },
    friendly: {
      subject: `Hello from [Your Company] – ${industry || 'Business'} solutions`,
      body: `Hi [Name],

I hope this message finds you well! I've been following ${industry || 'your industry'} for a while and really admired what you've built at [Company].

We recently worked with a company facing similar challenges as yours, and the results were fantastic. I thought you might be interested in learning how we could help streamline your operations.

No pressure at all – just wanted to introduce myself and see if there's a fit.

Warmly,
[Your Name]`
    },
    value: {
      subject: `3 ideas for ${rolePhrase}s in ${industry || 'your field'}`,
      body: `Hi [Name],

Most ${rolePhrase}s we speak with struggle with [common pain point in ${industry || 'this industry'}].

We've developed a solution that's helped over 50+ companies address this exact challenge, resulting in 20-30% cost reduction.

I'd love to share a quick case study that's relevant to your situation. Would that be helpful?

Cheers,
[Your Name]`
    }
  }

  return templates[style] || templates.direct
}

// 主生成函数
const generateMessages = async () => {
  generating.value = true
  messages.value = []

  try {
    let styles = []
    if (stylePref.value === 'balanced') {
      styles = ['direct', 'friendly', 'value']
    } else {
      styles = [stylePref.value, 'direct', 'friendly'].filter((v, i, a) => a.indexOf(v) === i).slice(0, 3)
    }

    for (const style of styles) {
      const msg = await generateSingle(style, chineseInput.value, industry.value, targetRole.value)
      messages.value.push(msg)
    }

    history.value.unshift({
      input: chineseInput.value,
      time: new Date().toLocaleString('zh-CN'),
      messages: JSON.parse(JSON.stringify(messages.value))
    })
    if (history.value.length > 10) history.value.pop()

  } catch (err) {
    alert('生成失败：' + err.message)
  } finally {
    generating.value = false
  }
}

// 重新生成单条
const regenerateOne = async (idx) => {
  const msg = messages.value[idx]
  messages.value[idx] = await generateSingle(msg.style, chineseInput.value, industry.value, targetRole.value)
}

// 复制
const copyMessage = (msg) => {
  const text = `Subject: ${msg.subject}\n\n${msg.body}`
  navigator.clipboard.writeText(text)
  alert('已复制到剪贴板！')
}

// 加载历史
const loadHistory = (h) => {
  chineseInput.value = h.input
  messages.value = h.messages || []
}

// 测试 API
const testAPI = async () => {
  try {
    apiStatus.value = { type: 'info', msg: '🔄 测试中...' }

    if (apiMode.value === 'mock') {
      apiStatus.value = { type: 'info', msg: 'ℹ️ 模拟模式无需测试，直接生成即可（使用内置模板）。' }
      return
    }

    if (apiMode.value === 'free') {
      if (!apiKey.value) {
        apiStatus.value = { type: 'error', msg: '❌ 请先填入 Gemini API Key' }
        return
      }
      // 测试 Gemini
      const url = `${GEMINI_ENDPOINT}?key=${apiKey.value}`
      const body = {
        contents: [{ parts: [{ text: 'Reply with just: OK' }] }],
        generationConfig: { maxOutputTokens: 10 }
      }
      const res = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
      })
      if (res.ok) {
        apiStatus.value = { type: 'success', msg: '✅ Gemini API 连接成功！可以生成开发信了。' }
      } else {
        const err = await res.json()
        apiStatus.value = { type: 'error', msg: `❌ 连接失败：${JSON.stringify(err)}` }
      }
      return
    }

    // 自定义模式测试
    if (!apiEndpoint.value) {
      apiStatus.value = { type: 'info', msg: 'ℹ️ 未配置 API Endpoint，将使用模拟模式。' }
      return
    }
    const body = {
      model: 'gpt-3.5-turbo',
      messages: [{ role: 'user', content: 'Reply with just: OK' }],
      max_tokens: 5,
    }
    const headers = { 'Content-Type': 'application/json' }
    if (apiKey.value) headers['Authorization'] = `Bearer ${apiKey.value}`

    const res = await fetch(apiEndpoint.value, {
      method: 'POST',
      headers,
      body: JSON.stringify(body),
    })
    if (res.ok) {
      apiStatus.value = { type: 'success', msg: '✅ API 连接成功！可以生成开发信了。' }
    } else {
      apiStatus.value = { type: 'error', msg: `❌ API 测试失败：${res.status} ${await res.text()}` }
    }
  } catch (err) {
    apiStatus.value = { type: 'error', msg: `❌ 连接失败：${err.message}` }
  }
}
</script>

<style scoped>
.outreach-writer {
  max-width: 900px;
  margin: 0 auto;
}
h2 {
  color: #0a66c2;
  margin-bottom: 4px;
}
.subtitle {
  color: #666;
  margin-bottom: 24px;
  font-size: 14px;
}

/* API Section */
.api-section {
  background: #f8f9fa;
  border-radius: 12px;
  margin-bottom: 24px;
  overflow: hidden;
  border: 2px solid #e0e0e0;
}
.api-tabs {
  display: flex;
  border-bottom: 2px solid #e0e0e0;
}
.api-tab {
  flex: 1;
  padding: 12px 16px;
  background: none;
  border: none;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  color: #666;
  transition: all 0.2s;
  border-bottom: 3px solid transparent;
  margin-bottom: -2px;
}
.api-tab.active {
  color: #0a66c2;
  border-bottom-color: #0a66c2;
  background: white;
}
.api-tab:hover {
  background: #eee;
}
.api-panel {
  padding: 20px;
}
.free-panel {
  background: white;
}
.mock-panel {
  background: #fff8e1;
}

/* Free steps */
.free-steps {
  display: flex;
  flex-direction: column;
  gap: 16px;
  margin-bottom: 16px;
}
.step {
  display: flex;
  gap: 12px;
  align-items: flex-start;
}
.step-num {
  width: 28px;
  height: 28px;
  background: #0a66c2;
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  font-weight: 700;
  flex-shrink: 0;
  margin-top: 2px;
}
.step-content {
  flex: 1;
}
.step-content strong {
  display: block;
  margin-bottom: 4px;
  font-size: 14px;
}
.step-content p {
  font-size: 13px;
  color: #555;
  margin: 0;
  line-height: 1.5;
}
.step-content a {
  color: #0a66c2;
  text-decoration: none;
  font-weight: 600;
}
.step-content a:hover {
  text-decoration: underline;
}
.key-input-row {
  display: flex;
  gap: 8px;
  margin-top: 6px;
}
.key-input-row input {
  flex: 1;
  padding: 8px 12px;
  border: 2px solid #0a66c2;
  border-radius: 6px;
  font-size: 13px;
}
.btn-test {
  background: #0a66c2;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
  font-weight: 600;
  white-space: nowrap;
}
.free-info {
  background: #e8f5e9;
  padding: 10px 14px;
  border-radius: 8px;
  font-size: 12px;
  color: #2e7d32;
  line-height: 1.5;
}
.mock-info {
  font-size: 13px;
  color: #f57f17;
  line-height: 1.6;
  margin: 0;
}

/* Custom API */
.api-row {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
}
.api-row input {
  flex: 1;
  min-width: 200px;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
}
.btn-link {
  background: none;
  border: 1px solid #0a66c2;
  color: #0a66c2;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
}
.api-msg.success { color: #2e7d32; font-size: 13px; margin-top: 10px; }
.api-msg.error { color: #d93025; font-size: 13px; margin-top: 10px; }
.api-msg.info { color: #666; font-size: 13px; margin-top: 10px; }

/* Input section */
.input-section {
  background: white;
  padding: 24px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  margin-bottom: 24px;
}
.input-section label {
  font-weight: 600;
  display: block;
  margin-bottom: 8px;
  font-size: 14px;
}
textarea {
  width: 100%;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 14px;
  resize: vertical;
  box-sizing: border-box;
  font-family: inherit;
}
textarea:focus {
  outline: none;
  border-color: #0a66c2;
}
.options-row {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 12px;
  margin: 16px 0;
}
.option-group input, .option-group select {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  box-sizing: border-box;
}
.btn-generate {
  background: linear-gradient(135deg, #0a66c2, #004182);
  color: white;
  border: none;
  padding: 14px 32px;
  border-radius: 8px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  width: 100%;
  transition: opacity 0.2s;
}
.btn-generate:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Results */
.results-section {
  margin-bottom: 24px;
}
.results-section h3 {
  margin-bottom: 16px;
}
.message-card {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
  margin-bottom: 16px;
  overflow: hidden;
  border-left: 4px solid #0a66c2;
}
.message-card.medium { border-left-color: #f0a500; }
.message-card.high { border-left-color: #d93025; }

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #f8f9fa;
  flex-wrap: wrap;
  gap: 8px;
}
.style-badge {
  font-weight: 700;
  font-size: 14px;
  color: #0a66c2;
}
.risk-badge {
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 12px;
  font-weight: 600;
}
.risk-badge.low { background: #e8f5e9; color: #2e7d32; }
.risk-badge.medium { background: #fff8e1; color: #f57f17; }
.risk-badge.high { background: #ffebee; color: #d93025; }
.message-content {
  padding: 16px;
}
.field {
  margin-bottom: 12px;
}
.field label {
  font-weight: 600;
  font-size: 12px;
  color: #666;
  display: block;
  margin-bottom: 4px;
}
.text-block {
  background: #f8f9fa;
  padding: 12px;
  border-radius: 6px;
  font-size: 14px;
  line-height: 1.6;
  white-space: pre-wrap;
  border: 1px solid #eee;
}
.card-actions {
  padding: 12px 16px;
  background: #f8f9fa;
  display: flex;
  gap: 8px;
  align-items: center;
  flex-wrap: wrap;
}
.btn-action {
  background: white;
  border: 1px solid #0a66c2;
  color: #0a66c2;
  padding: 6px 14px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 13px;
}
.spam-analysis {
  margin-left: auto;
  font-size: 12px;
  color: #666;
}
.spam-analysis .low { color: #2e7d32; font-weight: 600; }
.spam-analysis .medium { color: #f57f17; font-weight: 600; }
.spam-analysis .high { color: #d93025; font-weight: 600; }

/* History */
.history-section h3 {
  margin-bottom: 12px;
}
.history-item {
  background: white;
  padding: 12px 16px;
  border-radius: 8px;
  margin-bottom: 8px;
  cursor: pointer;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border: 1px solid #eee;
  transition: background 0.2s;
}
.history-item:hover {
  background: #f8f9fa;
}
.history-preview {
  font-size: 13px;
  color: #333;
}
.history-time {
  font-size: 12px;
  color: #999;
}
</style>

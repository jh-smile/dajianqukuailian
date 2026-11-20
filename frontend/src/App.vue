<script setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import BlockCard from './components/BlockCard.vue'
import NodePanel from './components/NodePanel.vue'

const STORAGE_KEY = 'blockchain-demo-nodes'
const DEFAULT_PORTS = [3001]
const AUTO_REFRESH_INTERVAL = 1000

const nodes = ref([])
const activeBase = ref('')
const blocks = ref([])
const chainError = ref('')
const chainLoading = ref(false)
const nodesLoading = ref(false)
const autoRefresh = ref(true)
const activeSection = ref('overview')
let refreshTimer = null

const activeNode = computed(
  () => nodes.value.find((node) => node.apiBase === activeBase.value) || null,
)
const p2pPort = computed(() => activeNode.value?.p2pPort ?? null)
const connected = computed(() => Boolean(activeNode.value))

const latestBlock = computed(() => blocks.value[blocks.value.length - 1] || null)
const shortHash = computed(() => {
  const hash = latestBlock.value?.hash
  if (!hash) return '—'
  return hash.length > 22 ? `${hash.slice(0, 11)}…${hash.slice(-9)}` : hash
})

function buildBase(port) {
  const protocol = window.location.protocol === 'https:' ? 'https:' : 'http:'
  const host = window.location.hostname || 'localhost'
  return `${protocol}//${host}:${port}`
}

function readStoredPorts() {
  try {
    const stored = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]')
    if (!Array.isArray(stored)) return []
    return stored
      .map(Number)
      .filter((port) => Number.isInteger(port) && port > 0 && port < 65536)
  } catch {
    return []
  }
}

function storePorts() {
  const ports = [...new Set(nodes.value.map((node) => node.apiPort))]
  localStorage.setItem(STORAGE_KEY, JSON.stringify(ports))
}

async function fetchJson(url, timeout) {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), timeout)
  try {
    const response = await fetch(url, { signal: controller.signal })
    if (!response.ok) throw new Error(`HTTP ${response.status}`)
    return await response.json()
  } finally {
    clearTimeout(timeoutId)
  }
}

async function refreshNodes() {
  if (nodesLoading.value) return
  nodesLoading.value = true

  const candidatePorts = [
    ...new Set([
      ...readStoredPorts(),
      ...DEFAULT_PORTS,
      ...nodes.value.map((node) => node.apiPort),
    ]),
  ]

  const results = await Promise.all(
    candidatePorts.map(async (port) => {
      const apiBase = buildBase(port)
      try {
        const info = await fetchJson(`${apiBase}/nodeInfo`, 1500)
        return {
          apiPort: port,
          apiBase,
          p2pPort: info?.p2pPort ?? null,
          blockCount: Number(info?.blockCount) || 0,
        }
      } catch {
        return null
      }
    }),
  )

  nodes.value = results.filter(Boolean)
  storePorts()
  nodesLoading.value = false

  const stillOnline = nodes.value.some((node) => node.apiBase === activeBase.value)
  if (!stillOnline) {
    const fallback = nodes.value[0] || null
    activeBase.value = fallback?.apiBase || ''
    blocks.value = []
    if (fallback) {
      await loadChain()
    } else {
      chainError.value = '未发现可用节点，请先在后端执行 api <端口> 启动 HTTP 服务'
    }
  }
}

async function loadChain() {
  if (!activeBase.value) return
  chainLoading.value = true
  try {
    const data = await fetchJson(`${activeBase.value}/blocks`, 5000)
    blocks.value = Array.isArray(data) ? data : []
    chainError.value = ''
    const node = nodes.value.find((item) => item.apiBase === activeBase.value)
    if (node) node.blockCount = blocks.value.length
  } catch (error) {
    chainError.value = `无法获取 ${activeBase.value} 的区块链数据（${error.message}）`
  } finally {
    chainLoading.value = false
  }
}

async function selectNode(node) {
  if (node.apiBase === activeBase.value) return
  activeBase.value = node.apiBase
  blocks.value = []
  await loadChain()
}

async function addNode(port) {
  const apiBase = buildBase(port)
  try {
    const info = await fetchJson(`${apiBase}/nodeInfo`, 2000)
    const node = {
      apiPort: port,
      apiBase,
      p2pPort: info?.p2pPort ?? null,
      blockCount: Number(info?.blockCount) || 0,
    }
    const existingIndex = nodes.value.findIndex((item) => item.apiBase === apiBase)
    if (existingIndex >= 0) {
      nodes.value[existingIndex] = node
    } else {
      nodes.value.push(node)
    }
    storePorts()
    activeBase.value = apiBase
    await loadChain()
  } catch {
    chainError.value = `无法连接节点 ${apiBase}，请确认该端口已启动 api 服务`
  }
}

function startTimer() {
  stopTimer()
  refreshTimer = setInterval(() => {
    if (!chainLoading.value) loadChain()
  }, AUTO_REFRESH_INTERVAL)
}

function stopTimer() {
  if (refreshTimer) {
    clearInterval(refreshTimer)
    refreshTimer = null
  }
}

watch(autoRefresh, (enabled) => {
  if (enabled) {
    startTimer()
  } else {
    stopTimer()
  }
})

onMounted(async () => {
  await refreshNodes()
  if (autoRefresh.value) startTimer()
})

onUnmounted(stopTimer)
</script>

<template>
  <div class="app-shell">
    <div class="ambient" aria-hidden="true"></div>

    <aside class="sidebar">
      <a class="brand" href="#overview" @click="activeSection = 'overview'">
        <span class="brand-mark" aria-hidden="true">
          <svg viewBox="0 0 24 24" width="22" height="22" fill="none">
            <path
              d="M12 2.6 20.4 7v10L12 21.4 3.6 17V7L12 2.6Z"
              stroke="currentColor"
              stroke-width="1.6"
              stroke-linejoin="round"
            />
            <path
              d="M3.9 7.2 12 11.6l8.1-4.4M12 11.6V21"
              stroke="currentColor"
              stroke-width="1.4"
              stroke-linejoin="round"
            />
          </svg>
        </span>
        <span class="brand-text">
          <span class="brand-name">ChainViz</span>
          <span class="brand-sub">Blockchain Dashboard</span>
        </span>
      </a>

      <nav class="side-nav" aria-label="页面导航">
        <a
          href="#overview"
          class="nav-item"
          :class="{ active: activeSection === 'overview' }"
          @click="activeSection = 'overview'"
        >
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" aria-hidden="true">
            <rect x="4" y="4" width="7" height="7" rx="1.6" stroke="currentColor" stroke-width="1.7" />
            <rect x="13" y="4" width="7" height="7" rx="1.6" stroke="currentColor" stroke-width="1.7" />
            <rect x="4" y="13" width="7" height="7" rx="1.6" stroke="currentColor" stroke-width="1.7" />
            <rect x="13" y="13" width="7" height="7" rx="1.6" stroke="currentColor" stroke-width="1.7" />
          </svg>
          概览
        </a>
        <a
          href="#nodes"
          class="nav-item"
          :class="{ active: activeSection === 'nodes' }"
          @click="activeSection = 'nodes'"
        >
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" aria-hidden="true">
            <circle cx="6" cy="6" r="2.5" stroke="currentColor" stroke-width="1.7" />
            <circle cx="18" cy="18" r="2.5" stroke="currentColor" stroke-width="1.7" />
            <path d="M6 8.5v4a3 3 0 0 0 3 3h6" stroke="currentColor" stroke-width="1.7" />
          </svg>
          节点
        </a>
        <a
          href="#blocks"
          class="nav-item"
          :class="{ active: activeSection === 'blocks' }"
          @click="activeSection = 'blocks'"
        >
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" aria-hidden="true">
            <rect x="4" y="4" width="16" height="6" rx="2" stroke="currentColor" stroke-width="1.7" />
            <rect x="4" y="14" width="16" height="6" rx="2" stroke="currentColor" stroke-width="1.7" />
          </svg>
          区块流
        </a>
      </nav>

      <div class="side-footer">
        <div class="side-status" :class="connected ? 'online' : 'offline'">
          <span class="side-dot"></span>
          <span>{{ connected ? '网络已连接' : '网络未连接' }}</span>
        </div>
        <span class="side-meta">Blockchain Demo · v1.0</span>
      </div>
    </aside>

    <main class="main">
      <div class="main-inner">
        <header class="topbar">
          <div>
            <h1>区块链仪表盘</h1>
            <p>实时同步后端链上数据与节点状态</p>
          </div>

          <div class="network-card">
            <div class="network-item">
              <span class="network-label">P2P</span>
              <span class="network-value">{{ p2pPort ?? '—' }}</span>
            </div>
            <span class="network-divider"></span>
            <div class="network-item">
              <span class="network-label">API</span>
              <span class="network-value">:{{ activeNode?.apiPort ?? '—' }}</span>
            </div>
            <span class="status-chip" :class="connected ? 'online' : 'offline'">
              <span class="status-dot"></span>
              {{ connected ? '已连接' : '未连接' }}
            </span>
          </div>
        </header>

        <section id="overview" class="stats" aria-label="区块链概览">
          <article class="stat-card">
            <div class="stat-top">
              <span class="stat-icon purple" aria-hidden="true">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none">
                  <path
                    d="M12 2.6 20.4 7v10L12 21.4 3.6 17V7L12 2.6Z"
                    stroke="currentColor"
                    stroke-width="1.7"
                    stroke-linejoin="round"
                  />
                  <path
                    d="M3.9 7.2 12 11.6l8.1-4.4M12 11.6V21"
                    stroke="currentColor"
                    stroke-width="1.5"
                    stroke-linejoin="round"
                  />
                </svg>
              </span>
              <span class="stat-label">区块总数</span>
            </div>
            <strong class="stat-value">{{ blocks.length }}</strong>
            <span class="stat-foot">当前链上区块</span>
          </article>

          <article class="stat-card">
            <div class="stat-top">
              <span class="stat-icon blue" aria-hidden="true">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none">
                  <path
                    d="m12 3 9 5-9 5-9-5 9-5Z"
                    stroke="currentColor"
                    stroke-width="1.7"
                    stroke-linejoin="round"
                  />
                  <path
                    d="m3 12 9 5 9-5M3 16l9 5 9-5"
                    stroke="currentColor"
                    stroke-width="1.7"
                    stroke-linejoin="round"
                  />
                </svg>
              </span>
              <span class="stat-label">最新区块</span>
            </div>
            <strong class="stat-value">{{ latestBlock ? `#${latestBlock.index}` : '—' }}</strong>
            <span class="stat-foot">链上最新高度</span>
          </article>

          <article class="stat-card">
            <div class="stat-top">
              <span class="stat-icon orange" aria-hidden="true">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none">
                  <path
                    d="M9.5 4 7.5 20M16.5 4l-2 16M4 9h16M3 15h16"
                    stroke="currentColor"
                    stroke-width="1.7"
                    stroke-linecap="round"
                  />
                </svg>
              </span>
              <span class="stat-label">最新哈希</span>
            </div>
            <strong class="stat-value mono">{{ shortHash }}</strong>
            <span class="stat-foot">SHA-256 指纹</span>
          </article>

          <article class="stat-card">
            <div class="stat-top">
              <span class="stat-icon green" aria-hidden="true">
                <svg viewBox="0 0 24 24" width="16" height="16" fill="none">
                  <circle cx="6" cy="6" r="2.5" stroke="currentColor" stroke-width="1.7" />
                  <circle cx="18" cy="18" r="2.5" stroke="currentColor" stroke-width="1.7" />
                  <path d="M6 8.5v4a3 3 0 0 0 3 3h6" stroke="currentColor" stroke-width="1.7" />
                </svg>
              </span>
              <span class="stat-label">在线节点</span>
            </div>
            <strong class="stat-value">{{ nodes.length }}</strong>
            <span class="stat-foot">已发现 API 节点</span>
          </article>
        </section>

        <section id="nodes" class="section-block">
          <div class="section-head">
            <h2>节点状态</h2>
            <span>选择要查看的节点</span>
          </div>
          <NodePanel
            :nodes="nodes"
            :active-base="activeBase"
            :refreshing="nodesLoading"
            @select="selectNode"
            @refresh="refreshNodes"
            @add="addNode"
          />
        </section>

        <section id="blocks" class="section-block">
          <div class="blocks-head">
            <div class="blocks-head-left">
              <h2>区块流</h2>
              <span class="count-chip">共 {{ blocks.length }} 个区块</span>
            </div>

            <div class="blocks-actions">
              <button type="button" class="btn-primary" :disabled="!activeBase" @click="loadChain">
                <svg viewBox="0 0 24 24" width="15" height="15" aria-hidden="true">
                  <path
                    fill="currentColor"
                    d="M17.65 6.35A7.95 7.95 0 0 0 12 4a8 8 0 1 0 7.73 10h-2.08A6 6 0 1 1 12 6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35Z"
                  />
                </svg>
                Refresh Chain
              </button>

              <label class="switch">
                <input v-model="autoRefresh" type="checkbox" />
                <span class="switch-track"><span class="switch-thumb"></span></span>
                <span class="switch-text">Auto refresh (1s)</span>
              </label>
            </div>
          </div>

          <div v-if="chainError" class="chain-error" role="alert">
            <svg viewBox="0 0 24 24" width="18" height="18" fill="none" aria-hidden="true">
              <circle cx="12" cy="12" r="9" stroke="currentColor" stroke-width="1.7" />
              <path
                d="M12 7.5v5.5M12 16.5h.01"
                stroke="currentColor"
                stroke-width="1.8"
                stroke-linecap="round"
              />
            </svg>
            <span>{{ chainError }}</span>
          </div>

          <div v-if="blocks.length" class="chain">
            <div v-for="block in blocks" :key="block.hash" class="timeline-item">
              <span class="timeline-dot" aria-hidden="true"></span>
              <BlockCard :block="block" />
            </div>
          </div>

          <div v-else-if="!chainError" class="chain-empty">
            <span v-if="chainLoading" class="spinner"></span>
            <span v-else class="empty-icon" aria-hidden="true">
              <svg viewBox="0 0 24 24" width="22" height="22" fill="none">
                <rect x="4" y="4" width="16" height="16" rx="4" stroke="currentColor" stroke-width="1.6" />
                <path
                  d="M8 9h8M8 12h8M8 15h5"
                  stroke="currentColor"
                  stroke-width="1.6"
                  stroke-linecap="round"
                />
              </svg>
            </span>
            <p>{{ chainLoading ? '正在加载区块…' : '暂无区块数据' }}</p>
          </div>
        </section>
      </div>
    </main>
  </div>
</template>

<style scoped>
.app-shell {
  position: relative;
  min-height: 100vh;
}

.ambient {
  position: fixed;
  inset: 0;
  z-index: -1;
  background:
    radial-gradient(900px 520px at 12% -10%, rgba(139, 123, 255, 0.18), transparent 62%),
    radial-gradient(760px 480px at 95% 4%, rgba(61, 149, 238, 0.12), transparent 60%),
    radial-gradient(900px 600px at 50% 115%, rgba(109, 91, 240, 0.1), transparent 60%),
    var(--bg);
}

.ambient::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image:
    linear-gradient(rgba(255, 255, 255, 0.022) 1px, transparent 1px),
    linear-gradient(90deg, rgba(255, 255, 255, 0.022) 1px, transparent 1px);
  background-size: 44px 44px;
  -webkit-mask-image: radial-gradient(ellipse at 50% 0%, #000 0%, transparent 72%);
  mask-image: radial-gradient(ellipse at 50% 0%, #000 0%, transparent 72%);
  pointer-events: none;
}

.sidebar {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  z-index: 20;
  display: flex;
  flex-direction: column;
  gap: 28px;
  width: 236px;
  padding: 26px 18px;
  border-right: 1px solid var(--line);
  background: rgba(11, 14, 20, 0.82);
  backdrop-filter: blur(14px);
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
  text-decoration: none;
}

.brand-mark {
  display: grid;
  place-items: center;
  flex: 0 0 auto;
  width: 44px;
  height: 44px;
  border-radius: 13px;
  background: linear-gradient(135deg, #9a8bff 0%, #6d5bf0 100%);
  color: #fff;
  box-shadow:
    0 10px 22px rgba(109, 91, 240, 0.38),
    inset 0 1px 0 rgba(255, 255, 255, 0.3);
}

.brand-text {
  display: flex;
  flex-direction: column;
  min-width: 0;
}

.brand-name {
  color: #fff;
  font-size: 18px;
  font-weight: 750;
  letter-spacing: 0.2px;
  line-height: 1.2;
}

.brand-sub {
  color: #6f788c;
  font-size: 9.5px;
  letter-spacing: 0.9px;
  text-transform: uppercase;
  white-space: nowrap;
}

.side-nav {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.nav-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border: 1px solid transparent;
  border-radius: 10px;
  color: #9aa3b8;
  font-size: 13.5px;
  text-decoration: none;
  transition:
    background 0.18s ease,
    color 0.18s ease,
    border-color 0.18s ease;
}

.nav-item svg {
  opacity: 0.8;
}

.nav-item:hover {
  border-color: var(--line);
  background: rgba(255, 255, 255, 0.04);
  color: #d9def0;
}

.nav-item.active {
  border-color: rgba(139, 123, 255, 0.24);
  background: linear-gradient(90deg, rgba(139, 123, 255, 0.18), rgba(139, 123, 255, 0.04));
  color: #e4dfff;
  box-shadow: inset 2px 0 0 var(--brand);
}

.side-footer {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: auto;
  padding-top: 16px;
  border-top: 1px solid var(--line);
}

.side-status {
  display: flex;
  align-items: center;
  gap: 7px;
  font-size: 12.5px;
  color: var(--green);
}

.side-status.offline {
  color: #ff7b7b;
}

.side-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: currentColor;
}

.side-status.online .side-dot {
  animation: status-pulse 1.9s ease-out infinite;
}

.side-meta {
  color: #5f6779;
  font-size: 10.5px;
}

.main {
  margin-left: 236px;
  padding: 34px 36px 80px;
}

.main-inner {
  max-width: 1070px;
  margin: 0 auto;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 18px;
  margin-bottom: 26px;
}

.topbar h1 {
  margin: 0;
  color: #fff;
  font-size: 26px;
  font-weight: 750;
  letter-spacing: -0.3px;
  line-height: 1.2;
}

.topbar p {
  margin: 3px 0 0;
  color: var(--muted);
  font-size: 12.5px;
}

.network-card {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 10px 14px;
  border: 1px solid var(--line);
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.035);
  backdrop-filter: blur(8px);
}

.network-item {
  display: flex;
  flex-direction: column;
  line-height: 1.3;
}

.network-label {
  color: #6f788c;
  font-size: 9.5px;
  letter-spacing: 0.8px;
  text-transform: uppercase;
}

.network-value {
  color: #dfe4f2;
  font-size: 13.5px;
  font-weight: 650;
  font-variant-numeric: tabular-nums;
}

.network-divider {
  width: 1px;
  height: 26px;
  background: var(--line);
}

.status-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 5px 10px;
  border: 1px solid rgba(61, 220, 151, 0.25);
  border-radius: 999px;
  background: rgba(61, 220, 151, 0.1);
  color: var(--green);
  font-size: 12px;
  font-weight: 600;
  white-space: nowrap;
}

.status-chip.offline {
  border-color: rgba(255, 107, 107, 0.28);
  background: rgba(255, 107, 107, 0.1);
  color: #ff7b7b;
}

.status-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}

.status-chip.online .status-dot {
  animation: status-pulse 1.9s ease-out infinite;
}

@keyframes status-pulse {
  0% {
    box-shadow: 0 0 0 0 rgba(61, 220, 151, 0.45);
  }
  70% {
    box-shadow: 0 0 0 6px rgba(61, 220, 151, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(61, 220, 151, 0);
  }
}

.stats {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 14px;
}

.stat-card {
  position: relative;
  min-width: 0;
  padding: 16px 18px 17px;
  border: 1px solid var(--line);
  border-radius: 15px;
  background: linear-gradient(165deg, rgba(255, 255, 255, 0.055), rgba(255, 255, 255, 0.02));
  box-shadow: 0 14px 30px rgba(0, 0, 0, 0.2);
  overflow: hidden;
  transition:
    transform 0.2s ease,
    border-color 0.2s ease;
}

.stat-card:hover {
  transform: translateY(-2px);
  border-color: rgba(139, 123, 255, 0.28);
}

.stat-card::after {
  content: '';
  position: absolute;
  top: -52px;
  right: -42px;
  width: 150px;
  height: 110px;
  border-radius: 50%;
  background: radial-gradient(circle, rgba(139, 123, 255, 0.22), transparent 70%);
  filter: blur(6px);
  pointer-events: none;
}

.stat-card:nth-child(2)::after {
  background: radial-gradient(circle, rgba(61, 149, 238, 0.2), transparent 70%);
}

.stat-card:nth-child(3)::after {
  background: radial-gradient(circle, rgba(242, 164, 95, 0.18), transparent 70%);
}

.stat-card:nth-child(4)::after {
  background: radial-gradient(circle, rgba(61, 220, 151, 0.16), transparent 70%);
}

.stat-top {
  display: flex;
  align-items: center;
  gap: 9px;
}

.stat-icon {
  display: grid;
  place-items: center;
  width: 30px;
  height: 30px;
  border-radius: 9px;
}

.stat-icon.purple {
  background: rgba(139, 123, 255, 0.14);
  color: #b5aaff;
}

.stat-icon.blue {
  background: rgba(61, 149, 238, 0.14);
  color: #7fbdf7;
}

.stat-icon.orange {
  background: rgba(242, 164, 95, 0.13);
  color: var(--orange);
}

.stat-icon.green {
  background: rgba(61, 220, 151, 0.12);
  color: #5be6ad;
}

.stat-label {
  color: var(--muted);
  font-size: 12px;
}

.stat-value {
  display: block;
  margin-top: 10px;
  color: #fff;
  font-size: 25px;
  font-weight: 750;
  letter-spacing: -0.3px;
  line-height: 1.2;
}

.stat-value.mono {
  overflow: hidden;
  color: var(--orange);
  font-family: ui-monospace, Consolas, 'Courier New', monospace;
  font-size: 14px;
  font-weight: 600;
  letter-spacing: 0;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.stat-foot {
  display: block;
  margin-top: 5px;
  color: #646c7f;
  font-size: 11px;
}

.section-block {
  scroll-margin-top: 24px;
}

.section-head {
  display: flex;
  align-items: baseline;
  gap: 10px;
  margin: 28px 0 12px;
}

.section-head h2 {
  margin: 0;
  color: #eef1f8;
  font-size: 17px;
  font-weight: 650;
}

.section-head span {
  color: var(--muted);
  font-size: 12px;
}

.blocks-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 14px;
  margin: 30px 0 18px;
}

.blocks-head-left {
  display: flex;
  align-items: center;
  gap: 10px;
}

.blocks-head h2 {
  margin: 0;
  color: #eef1f8;
  font-size: 17px;
  font-weight: 650;
}

.count-chip {
  padding: 4px 10px;
  border: 1px solid rgba(139, 123, 255, 0.18);
  border-radius: 999px;
  background: rgba(139, 123, 255, 0.1);
  color: #b8b0ff;
  font-size: 11.5px;
  white-space: nowrap;
}

.blocks-actions {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 14px;
}

.btn-primary {
  display: inline-flex;
  align-items: center;
  gap: 7px;
  padding: 9px 16px;
  border: none;
  border-radius: 10px;
  background: linear-gradient(135deg, #4aa1f4 0%, #2f7fe0 100%);
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  box-shadow: 0 8px 18px rgba(47, 134, 224, 0.28);
  transition:
    transform 0.18s ease,
    box-shadow 0.18s ease;
}

.btn-primary:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 11px 22px rgba(47, 134, 224, 0.4);
}

.switch {
  display: inline-flex;
  align-items: center;
  gap: 9px;
  cursor: pointer;
  user-select: none;
}

.switch input {
  position: absolute;
  width: 0;
  height: 0;
  opacity: 0;
}

.switch-track {
  display: flex;
  align-items: center;
  width: 38px;
  height: 22px;
  padding: 3px;
  border: 1px solid var(--line);
  border-radius: 999px;
  background: #262d3c;
  transition: background 0.2s ease;
}

.switch-thumb {
  display: block;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: #dfe4f2;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.4);
  transition: transform 0.2s ease;
}

.switch input:checked + .switch-track {
  border-color: rgba(139, 123, 255, 0.5);
  background: linear-gradient(135deg, #8b7bff 0%, #5b5ef0 100%);
}

.switch input:checked + .switch-track .switch-thumb {
  transform: translateX(16px);
}

.switch input:focus-visible + .switch-track {
  outline: 2px solid var(--brand);
  outline-offset: 2px;
}

.switch-text {
  color: var(--muted);
  font-size: 12.5px;
}

.chain-error {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  margin: 0 0 18px;
  padding: 12px 15px;
  border: 1px solid rgba(255, 107, 107, 0.28);
  border-radius: 12px;
  background: rgba(255, 107, 107, 0.08);
  color: #ff9090;
  font-size: 13px;
  line-height: 1.5;
  word-break: break-word;
}

.chain-error svg {
  flex: 0 0 auto;
  margin-top: 1px;
}

.chain {
  position: relative;
  max-width: 820px;
  padding-left: 32px;
}

.chain::before {
  content: '';
  position: absolute;
  top: 14px;
  bottom: 14px;
  left: 8px;
  width: 2px;
  border-radius: 2px;
  background: linear-gradient(
    180deg,
    rgba(139, 123, 255, 0.55),
    rgba(61, 149, 238, 0.24) 55%,
    rgba(139, 123, 255, 0.08)
  );
}

.timeline-item {
  position: relative;
  margin-bottom: 18px;
}

.timeline-item:last-child {
  margin-bottom: 0;
}

.timeline-dot {
  position: absolute;
  top: 21px;
  left: -30px;
  width: 14px;
  height: 14px;
  border: 2px solid var(--brand);
  border-radius: 50%;
  background: var(--bg);
  box-shadow:
    0 0 0 3px rgba(139, 123, 255, 0.12),
    0 0 14px rgba(139, 123, 255, 0.55);
}

.chain-empty {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  padding: 46px 0;
  color: var(--muted);
  font-size: 13px;
}

.chain-empty p {
  margin: 0;
}

.empty-icon {
  display: grid;
  place-items: center;
  width: 44px;
  height: 44px;
  border-radius: 14px;
  border: 1px solid var(--line);
  background: rgba(255, 255, 255, 0.04);
  color: #7c8498;
}

.spinner {
  width: 20px;
  height: 20px;
  border: 2px solid rgba(255, 255, 255, 0.12);
  border-top-color: var(--brand);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 1100px) {
  .stats {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@media (max-width: 900px) {
  .sidebar {
    position: static;
    width: auto;
    flex-direction: row;
    align-items: center;
    flex-wrap: wrap;
    gap: 14px 18px;
    padding: 14px 18px;
    border-right: none;
    border-bottom: 1px solid var(--line);
  }

  .brand-sub {
    display: none;
  }

  .side-nav {
    flex-direction: row;
    margin-left: auto;
  }

  .nav-item {
    padding: 8px 11px;
  }

  .side-footer {
    display: none;
  }

  .main {
    margin-left: 0;
    padding: 24px 18px 60px;
  }
}

@media (max-width: 600px) {
  .topbar h1 {
    font-size: 22px;
  }

  .network-card {
    width: 100%;
    justify-content: space-between;
  }

  .brand {
    width: 100%;
  }

  .side-nav {
    width: 100%;
    margin-left: 0;
    justify-content: space-between;
  }

  .blocks-actions {
    width: 100%;
    justify-content: space-between;
  }
}

@media (max-width: 440px) {
  .stats {
    grid-template-columns: minmax(0, 1fr);
  }
}
</style>

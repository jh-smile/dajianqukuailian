<script setup>
import { ref } from 'vue'

defineProps({
  nodes: {
    type: Array,
    default: () => [],
  },
  activeBase: {
    type: String,
    default: '',
  },
  refreshing: {
    type: Boolean,
    default: false,
  },
})

const emit = defineEmits(['select', 'refresh', 'add'])

const newPort = ref('')

function submitAdd() {
  const port = Number(newPort.value.trim())
  if (!Number.isInteger(port) || port < 1 || port > 65535) {
    return
  }
  emit('add', port)
  newPort.value = ''
}
</script>

<template>
  <section class="node-panel">
    <div class="panel-head">
      <span class="panel-icon" aria-hidden="true">
        <svg viewBox="0 0 24 24" width="15" height="15" fill="none">
          <circle cx="6" cy="6" r="3" stroke="currentColor" stroke-width="1.7" />
          <circle cx="18" cy="18" r="3" stroke="currentColor" stroke-width="1.7" />
          <path d="M6 9v4a3 3 0 0 0 3 3h6" stroke="currentColor" stroke-width="1.7" />
        </svg>
      </span>
      <span class="panel-label">可用节点</span>
    </div>

    <div class="node-list">
      <button
        v-for="node in nodes"
        :key="node.apiBase"
        type="button"
        class="node-chip"
        :class="{ active: node.apiBase === activeBase }"
        @click="emit('select', node)"
      >
        <span class="node-name">节点 {{ node.p2pPort || node.apiPort }}</span>
        <span class="node-blocks">{{ node.blockCount }} 区块</span>
      </button>
      <span v-if="nodes.length === 0" class="node-empty">暂无可用节点</span>
    </div>

    <div class="panel-actions">
      <input
        v-model="newPort"
        class="port-input"
        type="text"
        inputmode="numeric"
        placeholder="API 端口"
        aria-label="要添加的 API 端口"
        @keyup.enter="submitAdd"
      />
      <button type="button" class="btn-ghost" @click="submitAdd">添加</button>
      <button type="button" class="btn-refresh" :disabled="refreshing" @click="emit('refresh')">
        <svg viewBox="0 0 24 24" width="15" height="15" aria-hidden="true">
          <path
            fill="currentColor"
            d="M17.65 6.35A7.95 7.95 0 0 0 12 4a8 8 0 1 0 7.73 10h-2.08A6 6 0 1 1 12 6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35Z"
          />
        </svg>
        {{ refreshing ? '刷新中…' : '刷新节点' }}
      </button>
    </div>
  </section>
</template>

<style scoped>
.node-panel {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 14px 18px;
  padding: 16px 18px;
  background: linear-gradient(165deg, #161c2b 0%, #111725 100%);
  border: 1px solid var(--line);
  border-radius: var(--radius);
  box-shadow: 0 12px 28px rgba(0, 0, 0, 0.22);
}

.panel-head {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: var(--muted);
  font-size: 13px;
  white-space: nowrap;
}

.panel-icon {
  display: grid;
  place-items: center;
  width: 26px;
  height: 26px;
  border-radius: 8px;
  background: rgba(139, 123, 255, 0.14);
  color: #b7adff;
}

.node-list {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
  flex: 1 1 auto;
}

.node-chip {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  min-width: 98px;
  padding: 9px 16px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 12px;
  background: linear-gradient(180deg, #262d3d 0%, #1d2432 100%);
  color: #d6dceb;
  transition:
    transform 0.18s ease,
    box-shadow 0.18s ease,
    border-color 0.18s ease;
}

.node-chip:hover {
  transform: translateY(-2px);
  border-color: rgba(139, 123, 255, 0.32);
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.3);
}

.node-chip.active {
  border-color: rgba(139, 123, 255, 0.6);
  background: linear-gradient(135deg, #8b7bff 0%, #6d5bf0 100%);
  color: #fff;
  box-shadow:
    0 8px 20px rgba(109, 91, 240, 0.4),
    inset 0 1px 0 rgba(255, 255, 255, 0.24);
}

.node-name {
  font-size: 13px;
  font-weight: 650;
  white-space: nowrap;
}

.node-blocks {
  font-size: 11.5px;
  opacity: 0.78;
  white-space: nowrap;
}

.node-empty {
  color: var(--muted);
  font-size: 13px;
}

.panel-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-left: auto;
}

.port-input {
  width: 96px;
  padding: 8px 11px;
  border: 1px solid var(--line-strong);
  border-radius: 9px;
  background: rgba(9, 12, 19, 0.75);
  color: var(--text);
  font-size: 13px;
  transition:
    border-color 0.18s ease,
    box-shadow 0.18s ease;
}

.port-input::placeholder {
  color: #636c80;
}

.port-input:focus {
  outline: none;
  border-color: rgba(139, 123, 255, 0.7);
  box-shadow: 0 0 0 3px rgba(139, 123, 255, 0.16);
}

.btn-ghost {
  padding: 8px 12px;
  border: 1px solid rgba(61, 149, 238, 0.4);
  border-radius: 9px;
  background: rgba(61, 149, 238, 0.06);
  color: #7fbdf7;
  font-size: 13px;
  transition:
    background 0.18s ease,
    border-color 0.18s ease;
}

.btn-ghost:hover {
  border-color: rgba(61, 149, 238, 0.7);
  background: rgba(61, 149, 238, 0.14);
}

.btn-refresh {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 8px 15px;
  border: none;
  border-radius: 9px;
  background: linear-gradient(135deg, #4aa1f4 0%, #2f7fe0 100%);
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  box-shadow: 0 8px 18px rgba(47, 134, 224, 0.28);
  transition:
    transform 0.18s ease,
    box-shadow 0.18s ease;
}

.btn-refresh:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 10px 22px rgba(47, 134, 224, 0.4);
}

@media (max-width: 820px) {
  .panel-actions {
    width: 100%;
    margin-left: 0;
  }

  .port-input {
    flex: 1;
    width: auto;
  }
}
</style>

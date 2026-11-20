<script setup>
import { computed } from 'vue'

const props = defineProps({
  block: {
    type: Object,
    required: true,
  },
})

const formattedTime = computed(() => {
  const timestamp = Number(props.block.timestamp)
  if (!Number.isFinite(timestamp)) return '—'
  const date = new Date(timestamp)
  if (Number.isNaN(date.getTime())) return '—'
  const pad = (value) => String(value).padStart(2, '0')
  return `${date.getFullYear()}-${pad(date.getMonth() + 1)}-${pad(date.getDate())} ${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`
})
</script>

<template>
  <article class="block-card">
    <header class="block-head">
      <span class="block-index">#{{ block.index }}</span>
      <span class="block-time">
        <svg viewBox="0 0 24 24" width="13" height="13" fill="none" aria-hidden="true">
          <circle cx="12" cy="12" r="8.5" stroke="currentColor" stroke-width="1.6" />
          <path
            d="M12 7.5V12l3 2"
            stroke="currentColor"
            stroke-width="1.6"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
        {{ formattedTime }}
      </span>
    </header>

    <div class="block-body">
      <div class="row">
        <span class="row-label">Hash</span>
        <span class="row-value hash">{{ block.hash }}</span>
      </div>
      <div class="row">
        <span class="row-label">Prev Hash</span>
        <span class="row-value hash">{{ block.previousHash }}</span>
      </div>
      <div class="row">
        <span class="row-label">Data</span>
        <span class="row-value data">{{ block.data }}</span>
      </div>
      <div class="row">
        <span class="row-label">Nonce</span>
        <span class="row-value">
          <span class="nonce-badge">{{ block.nonce }}</span>
        </span>
      </div>
    </div>
  </article>
</template>

<style scoped>
.block-card {
  position: relative;
  background: linear-gradient(165deg, #161c2b 0%, #111725 100%);
  border: 1px solid var(--line);
  border-radius: var(--radius);
  overflow: hidden;
  box-shadow: 0 14px 32px rgba(0, 0, 0, 0.24);
  transition:
    transform 0.22s ease,
    box-shadow 0.22s ease,
    border-color 0.22s ease;
  animation: rise-in 0.35s ease both;
}

.block-card:hover {
  transform: translateY(-3px);
  border-color: rgba(139, 123, 255, 0.35);
  box-shadow:
    0 18px 40px rgba(0, 0, 0, 0.32),
    0 0 0 1px rgba(139, 123, 255, 0.12),
    0 0 26px rgba(139, 123, 255, 0.1);
}

.block-head {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 13px 20px;
  border-bottom: 1px solid var(--line);
  background:
    linear-gradient(120deg, rgba(139, 123, 255, 0.16) 0%, rgba(61, 149, 238, 0.06) 100%),
    rgba(255, 255, 255, 0.014);
  overflow: hidden;
}

.block-head::after {
  content: '';
  position: absolute;
  inset: 0;
  background: repeating-linear-gradient(
    120deg,
    rgba(255, 255, 255, 0.03) 0 1px,
    transparent 1px 16px
  );
  pointer-events: none;
}

.block-index {
  position: relative;
  z-index: 1;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 48px;
  padding: 3px 13px;
  border-radius: 999px;
  background: linear-gradient(135deg, #9a8bff 0%, #6d5bf0 100%);
  color: #fff;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: 0.4px;
  box-shadow: 0 4px 14px rgba(109, 91, 240, 0.42);
}

.block-time {
  position: relative;
  z-index: 1;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  color: var(--muted);
  font-size: 12.5px;
  font-variant-numeric: tabular-nums;
}

.block-body {
  padding: 16px 20px 18px;
}

.row {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 6px 0;
}

.row + .row {
  border-top: 1px solid rgba(255, 255, 255, 0.04);
}

.row-label {
  flex: 0 0 76px;
  padding-top: 6px;
  color: var(--muted);
  font-size: 12px;
  letter-spacing: 0.4px;
  text-transform: uppercase;
}

.row-value {
  flex: 1;
  min-width: 0;
  color: var(--text);
  font-size: 14px;
  overflow-wrap: anywhere;
}

.row-value.hash {
  display: block;
  padding: 6px 10px;
  border: 1px solid rgba(242, 164, 95, 0.18);
  border-radius: 8px;
  background: rgba(242, 164, 95, 0.07);
  color: var(--orange);
  font-family: ui-monospace, Consolas, 'Courier New', monospace;
  font-size: 12.5px;
  line-height: 1.5;
  word-break: break-all;
}

.row-value.data {
  padding-top: 5px;
  color: var(--text);
  font-weight: 600;
}

.nonce-badge {
  display: inline-block;
  margin-top: 3px;
  padding: 2px 10px;
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.05);
  color: #b9c1d3;
  font-size: 12.5px;
  font-weight: 600;
}

@keyframes rise-in {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@media (max-width: 560px) {
  .row {
    flex-direction: column;
    gap: 3px;
  }

  .row-label {
    flex: none;
    padding-top: 0;
  }
}
</style>

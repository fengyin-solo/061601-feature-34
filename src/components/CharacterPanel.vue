<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useGameStore } from '../stores/gameStore'
import gameConfig from '../config/gameConfig'
import {
  getAffinityColor,
  getAffinityStage,
  getMoodColor,
  getMoodLabel,
  getMoodEmoji,
  getMoodChangeReasonLabel,
  getMoodChangeReasonIcon,
  getMoodChangeColor,
  getTimeLabel
} from '../utils/gameUtils'
import type { MoodChangeRecord } from '../types/game'

const gameStore = useGameStore()

const animatedMoods = ref<Record<string, number>>({})
const moodFlash = ref<Record<string, boolean>>({})

const charactersWithConfig = computed(() => {
  return gameStore.unlockedCharacters.map(charState => {
    const config = gameConfig.characters.find(c => c.id === charState.id)
    return { state: charState, config }
  }).filter(item => item.config)
})

function selectCharacter(id: string) {
  gameStore.selectCharacter(id)
}

function formatChange(change: number): string {
  if (change > 0) return `+${change}`
  return `${change}`
}

const selectedCharacterMoodHistory = computed((): MoodChangeRecord[] => {
  if (!gameStore.selectedCharacterId) return []
  return gameStore.getRecentMoodChanges(gameStore.selectedCharacterId, 8)
})

watch(
  () => charactersWithConfig.value.map(c => ({ id: c.state.id, mood: c.state.mood })),
  (newVals, oldVals) => {
    const oldMap = new Map((oldVals || []).map(v => [v.id, v.mood]))
    newVals.forEach(({ id, mood }) => {
      const oldMood = oldMap.get(id)
      if (oldMood !== undefined && oldMood !== mood) {
        moodFlash.value[id] = true
        setTimeout(() => {
          moodFlash.value[id] = false
        }, 1000)
      }
      animatedMoods.value[id] = mood
    })
  },
  { immediate: true, deep: true }
)
</script>

<template>
  <div class="character-panel card">
    <h2 class="panel-title">
      <span class="title-icon">💕</span>
      角色状态
    </h2>

    <div class="character-list">
      <div
        v-for="item in charactersWithConfig"
        :key="item.state.id"
        class="character-card"
        :class="{ 
          selected: gameStore.selectedCharacterId === item.state.id,
          'mood-flash': moodFlash[item.state.id]
        }"
        @click="selectCharacter(item.state.id)"
      >
        <div class="character-avatar">
          <span class="avatar-emoji">{{ item.config?.avatar }}</span>
          <span 
            class="mood-badge" 
            :style="{ backgroundColor: getMoodColor(item.state.mood) }"
            :title="getMoodLabel(item.state.mood)"
          >
            {{ getMoodEmoji(item.state.mood) }}
          </span>
        </div>
        
        <div class="character-info">
          <div class="character-header">
            <span class="character-name">{{ item.config?.name }}</span>
            <span class="affinity-stage">{{ getAffinityStage(item.state.affinity) }}</span>
          </div>
          
          <div class="stat-row">
            <span class="stat-label">好感</span>
            <div class="progress-bar">
              <div 
                class="progress-fill"
                :style="{ 
                  width: `${Math.max(0, (item.state.affinity / gameConfig.maxAffinity) * 100)}%`,
                  backgroundColor: getAffinityColor(item.state.affinity, gameConfig.maxAffinity)
                }"
              ></div>
            </div>
            <span class="stat-value">{{ item.state.affinity }}</span>
          </div>
          
          <div class="stat-row mood-row">
            <span class="stat-label">心情</span>
            <div class="progress-bar mood-bar">
              <div 
                class="progress-fill"
                :style="{ 
                  width: `${(item.state.mood / gameConfig.maxMood) * 100}%`,
                  backgroundColor: getMoodColor(item.state.mood)
                }"
              ></div>
            </div>
            <span 
              class="stat-value mood-value" 
              :style="{ color: getMoodColor(item.state.mood) }"
            >
              <span class="mood-text">{{ getMoodLabel(item.state.mood) }}</span>
              <span class="mood-num">({{ item.state.mood }})</span>
            </span>
          </div>

          <div 
            v-if="item.state.lastMoodChangeReason"
            class="mood-change-hint"
            :class="{ 
              positive: item.state.lastMoodChange > 0, 
              negative: item.state.lastMoodChange < 0,
              neutral: item.state.lastMoodChange === 0
            }"
          >
            <span class="change-icon">{{ getMoodChangeReasonIcon(item.state.lastMoodChangeReason) }}</span>
            <span class="change-detail" :title="item.state.lastMoodChangeDetail">
              {{ getMoodChangeReasonLabel(item.state.lastMoodChangeReason) }}
            </span>
            <span 
              class="change-value"
              :style="{ color: getMoodChangeColor(item.state.lastMoodChange) }"
            >
              {{ formatChange(item.state.lastMoodChange) }}
            </span>
          </div>
        </div>
      </div>
    </div>

    <div v-if="gameStore.currentCharacterConfig" class="character-detail">
      <div class="detail-divider"></div>
      <p class="detail-description">{{ gameStore.currentCharacterConfig.description }}</p>
      <p class="detail-personality">性格：{{ gameStore.currentCharacterConfig.personality }}</p>

      <div v-if="selectedCharacterMoodHistory.length > 0" class="mood-history">
        <h3 class="history-title">
          <span class="history-icon">📊</span>
          心情变化记录
        </h3>
        <div class="history-list">
          <div 
            v-for="record in selectedCharacterMoodHistory"
            :key="record.id"
            class="history-item"
          >
            <div class="history-header">
              <span class="history-time">
                第{{ record.day }}天 {{ getTimeLabel(record.time) }}
              </span>
              <span 
                class="history-change"
                :style="{ color: getMoodChangeColor(record.change) }"
              >
                {{ formatChange(record.change) }}
              </span>
            </div>
            <div class="history-body">
              <span class="history-icon-sm">{{ getMoodChangeReasonIcon(record.reason) }}</span>
              <span class="history-reason">{{ getMoodChangeReasonLabel(record.reason) }}</span>
              <span 
                v-if="record.reasonDetail" 
                class="history-detail"
                :title="record.reasonDetail"
              >
                · {{ record.reasonDetail }}
              </span>
            </div>
            <div class="history-mood-bar">
              <div class="mood-range">
                <span class="mood-before" :style="{ color: getMoodColor(record.beforeMood) }">
                  {{ getMoodEmoji(record.beforeMood) }} {{ record.beforeMood }}
                </span>
                <span class="mood-arrow">→</span>
                <span class="mood-after" :style="{ color: getMoodColor(record.afterMood) }">
                  {{ getMoodEmoji(record.afterMood) }} {{ record.afterMood }}
                </span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div v-else class="mood-history empty">
        <h3 class="history-title">
          <span class="history-icon">📊</span>
          心情变化记录
        </h3>
        <p class="empty-hint">暂无心情变化记录，快去互动吧~</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.character-panel {
  padding: 20px;
}

.panel-title {
  font-size: 18px;
  font-weight: 600;
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.title-icon {
  font-size: 22px;
}

.character-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.character-card {
  display: flex;
  gap: 14px;
  padding: 14px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  cursor: pointer;
  transition: all 0.2s;
  border: 2px solid transparent;
  position: relative;
}

.character-card:hover {
  transform: translateX(4px);
  background: var(--accent-light);
}

.character-card.selected {
  border-color: var(--accent-primary);
  background: var(--accent-light);
}

.character-card.mood-flash {
  animation: moodFlash 1s ease-out;
}

@keyframes moodFlash {
  0% {
    box-shadow: 0 0 0 0 rgba(236, 72, 153, 0.4);
  }
  50% {
    box-shadow: 0 0 0 8px rgba(236, 72, 153, 0);
  }
  100% {
    box-shadow: 0 0 0 0 rgba(236, 72, 153, 0);
  }
}

.character-avatar {
  width: 56px;
  height: 56px;
  background: var(--bg-secondary);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
  flex-shrink: 0;
  position: relative;
}

.avatar-emoji {
  font-size: 28px;
}

.mood-badge {
  position: absolute;
  bottom: -4px;
  right: -4px;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  border: 2px solid var(--bg-tertiary);
  box-shadow: 0 2px 6px rgba(0,0,0,0.15);
}

.character-info {
  flex: 1;
  min-width: 0;
}

.character-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.character-name {
  font-weight: 600;
  font-size: 15px;
}

.affinity-stage {
  font-size: 12px;
  padding: 2px 8px;
  background: var(--accent-primary);
  color: white;
  border-radius: 9999px;
}

.stat-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.stat-label {
  font-size: 12px;
  color: var(--text-secondary);
  width: 30px;
  flex-shrink: 0;
}

.progress-bar {
  flex: 1;
  height: 6px;
  background: var(--bg-secondary);
  border-radius: 3px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 0.5s ease, background-color 0.3s ease;
}

.stat-value {
  font-size: 12px;
  font-weight: 500;
  min-width: 40px;
  text-align: right;
}

.mood-value {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  line-height: 1.1;
}

.mood-text {
  font-size: 11px;
  font-weight: 600;
}

.mood-num {
  font-size: 10px;
  color: var(--text-muted);
  opacity: 0.8;
}

.mood-change-hint {
  margin-top: 6px;
  padding: 4px 8px;
  background: var(--bg-secondary);
  border-radius: 6px;
  font-size: 11px;
  display: flex;
  align-items: center;
  gap: 4px;
  line-height: 1.4;
  border-left: 3px solid var(--border-color);
  animation: slideIn 0.3s ease-out;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.mood-change-hint.positive {
  border-left-color: #22c55e;
  background: #f0fdf4;
}

[data-theme='dark'] .mood-change-hint.positive {
  background: #14532d;
}

.mood-change-hint.negative {
  border-left-color: #ef4444;
  background: #fef2f2;
}

[data-theme='dark'] .mood-change-hint.negative {
  background: #7f1d1d;
}

.mood-change-hint.neutral {
  border-left-color: #6b7280;
  background: #f9fafb;
}

[data-theme='dark'] .mood-change-hint.neutral {
  background: #374151;
}

.change-icon {
  flex-shrink: 0;
}

.change-detail {
  flex: 1;
  color: var(--text-secondary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.change-value {
  font-weight: 600;
  flex-shrink: 0;
}

.character-detail {
  margin-top: 8px;
}

.detail-divider {
  height: 1px;
  background: var(--border-light);
  margin: 12px 0;
}

.detail-description {
  font-size: 13px;
  color: var(--text-secondary);
  line-height: 1.5;
  margin-bottom: 6px;
}

.detail-personality {
  font-size: 12px;
  color: var(--text-muted);
}

.mood-history {
  margin-top: 16px;
}

.mood-history.empty .empty-hint {
  text-align: center;
  color: var(--text-muted);
  padding: 20px;
  font-size: 13px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
}

.history-title {
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 10px;
  display: flex;
  align-items: center;
  gap: 6px;
  color: var(--text-primary);
}

.history-icon {
  font-size: 16px;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  max-height: 320px;
  overflow-y: auto;
  padding-right: 4px;
}

.history-item {
  padding: 10px 12px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-sm);
  border-left: 3px solid var(--border-color);
  animation: fadeIn 0.3s ease-out;
}

.history-item:nth-child(1) {
  border-left-color: #ec4899;
  background: linear-gradient(90deg, var(--accent-light) 0%, var(--bg-tertiary) 100%);
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateX(-8px);
  }
  to {
    opacity: 1;
    transform: translateX(0);
  }
}

.history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 4px;
}

.history-time {
  font-size: 11px;
  color: var(--text-muted);
}

.history-change {
  font-size: 12px;
  font-weight: 600;
}

.history-body {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: var(--text-secondary);
  margin-bottom: 6px;
  flex-wrap: wrap;
}

.history-icon-sm {
  flex-shrink: 0;
}

.history-reason {
  font-weight: 500;
  color: var(--text-primary);
}

.history-detail {
  color: var(--text-muted);
  font-size: 11px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 180px;
}

.history-mood-bar {
  padding-top: 4px;
  border-top: 1px dashed var(--border-light);
}

.mood-range {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  font-size: 11px;
}

.mood-before,
.mood-after {
  display: flex;
  align-items: center;
  gap: 2px;
  font-weight: 500;
}

.mood-arrow {
  color: var(--text-muted);
  flex-shrink: 0;
}
</style>

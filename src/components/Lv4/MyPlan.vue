<template>
  <div class="my-plan">
    <h2 class="section-title">🎯 マイプラン</h2>

    <!-- カード：目標と期間 -->
    <div class="card-section">
      <!-- 目標名入力 -->
      <div class="form-group">
        <label for="goal">目標名</label>
        <div class="goal-input-group">
          <input
            id="goal"
            v-model="planTitle"
            :disabled="!isEditingTitle"
            type="text"
            placeholder="例: 朝の散歩を習慣にする"
          />
          <button @click="openTemplateModal">テンプレートを見る</button>
          <button @click="toggleTitleEdit">
            {{ isEditingTitle ? '確定' : '目標名を変更' }}
          </button>
        </div>
      </div>

      <!-- 期間選択 -->
      <div class="form-group">
        <label for="duration">期間</label>
        <div class="duration-input-group">
          <select id="duration" v-model="selectedDuration" :disabled="isDurationLocked">
            <option disabled value="">選択してください</option>
            <option value="7">1週間</option>
            <option value="14">2週間</option>
            <option value="21">3週間</option>
            <option value="28">4週間</option>
          </select>
          <button v-if="selectedDuration" @click="toggleDurationLock">
            {{ isDurationLocked ? '期間を変更' : '確定' }}
          </button>
        </div>
        <p v-if="isDurationLocked && remainingDays !== null" class="remaining-days">
          期限まであと {{ remainingDays }} 日
        </p>
      </div>
    </div>

    <!-- カード：ステップ追加と進捗 -->
    <div class="card-section">
      <div class="step-section">
        <label>ステップを追加</label>
        <div class="step-input">
          <input v-model="newStep" placeholder="例: 毎朝7時に起きる" />
          <button @click="addStepAndSave" :disabled="!newStep.trim()">追加して保存</button>
        </div>

        <!-- 進捗バー -->
        <div class="progress-bar-wrapper" v-if="steps.length">
          <div class="progress-bar">
            <div class="progress" :style="{ width: progressPercent + '%' }"></div>
          </div>
          <p class="progress-text">{{ completedSteps }}/{{ steps.length }} ステップ完了</p>
        </div>

        <!-- ステップ一覧 -->
        <div class="step-cards">
          <div
            v-for="(step, index) in steps"
            :key="index"
            class="step-card"
            :class="{ completed: step.completed }"
          >
            <div class="step-card-inner" @click="openModal(step, index)">
              <span class="step-text">{{ step.text }}</span>
            </div>
            <input
              type="checkbox"
              class="step-checkbox"
              v-model="step.completed"
              @change="handleCheckboxChange($event)"
            />
          </div>
        </div>
      </div>
    </div>

    <!-- 詳細表示モーダル -->
    <div v-if="isModalOpen" class="modal-overlay" @click.self="closeModal">
      <div class="modal-content">
        <h3>ステップの詳細</h3>

        <template v-if="!isEditingStep">
          <p>{{ selectedStep?.text }}</p>
          <p>状態: {{ selectedStep?.completed ? '完了' : '未完了' }}</p>
          <button @click="isEditingStep = true">✏️ 編集する</button>
          <button @click="deleteSelectedStep">このステップを削除</button>
          <button @click="closeModal">閉じる</button>
        </template>

        <template v-else>
          <input v-model="editedText" class="edit-input" placeholder="ステップを編集..." />
          <div class="modal-buttons">
            <button @click="saveEditedStep">保存する</button>
            <button @click="isEditingStep = false">キャンセル</button>
          </div>
        </template>
      </div>
    </div>

    <!-- 目標テンプレート選択モーダル -->
    <div v-if="isTemplateModalOpen" class="modal-overlay" @click.self="closeTemplateModal">
      <div class="modal-content">
        <h3>目標テンプレート</h3>
        <ul class="template-list">
          <li
            v-for="template in goalTemplates"
            :key="template"
            class="template-item"
            @click="selectTemplate(template)"
          >
            {{ template }}
          </li>
        </ul>
        <button @click="closeTemplateModal">閉じる</button>
      </div>
    </div>

    <!-- トースト通知 -->
    <div v-if="showToast" class="toast-notification">
      ✅ 保存しました！
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const planTitle = ref('')
const isEditingTitle = ref(true)
const selectedDuration = ref('')
const newStep = ref('')
const steps = ref([])
const isModalOpen = ref(false)
const selectedStep = ref(null)
const selectedIndex = ref(null)
const isEditingStep = ref(false)
const editedText = ref('')
const showToast = ref(false)

const isDurationLocked = ref(false)
const planStartDate = ref(null)

const isTemplateModalOpen = ref(false)
const goalTemplates = ref([
  '朝の散歩を習慣にする',
  '毎日10分間ストレッチする',
  '寝る前に日記をつける',
  '週に2回は自炊する',
  '毎日水を2リットル飲む'
])

const openTemplateModal = () => {
  isTemplateModalOpen.value = true
}

const closeTemplateModal = () => {
  isTemplateModalOpen.value = false
}

const selectTemplate = (template) => {
  planTitle.value = template
  isEditingTitle.value = true // テンプレート選択後、編集モードで入力できるようにする
  closeTemplateModal()
}

const toggleTitleEdit = () => {
  isEditingTitle.value = !isEditingTitle.value
  if (!isEditingTitle.value) {
    savePlan()
  }
}

const toggleDurationLock = () => {
  isDurationLocked.value = !isDurationLocked.value
  if (isDurationLocked.value) {
    planStartDate.value = new Date()
  }
  savePlan()
}

const addStepAndSave = () => {
  steps.value.push({ text: newStep.value, completed: false })
  newStep.value = ''
  savePlan()
}

const openModal = (step, index) => {
  selectedStep.value = step
  selectedIndex.value = index
  isModalOpen.value = true
  isEditingStep.value = false
  editedText.value = step.text
}

const closeModal = () => {
  isModalOpen.value = false
  selectedStep.value = null
  selectedIndex.value = null
  isEditingStep.value = false
}

const deleteSelectedStep = () => {
  if (selectedIndex.value !== null) {
    steps.value.splice(selectedIndex.value, 1)
    savePlan()
    closeModal()
  }
}

const saveEditedStep = () => {
  if (selectedIndex.value !== null && editedText.value.trim()) {
    steps.value[selectedIndex.value].text = editedText.value.trim()
    savePlan()
    isEditingStep.value = false
  }
}

const handleCheckboxChange = (event) => {
  const checkbox = event.target
  checkbox.classList.add('animate')
  setTimeout(() => {
    checkbox.classList.remove('animate')
  }, 300)
  savePlan()
}

const completedSteps = computed(() => steps.value.filter((s) => s.completed).length)
const progressPercent = computed(() => {
  if (steps.value.length === 0) return 0
  return Math.round((completedSteps.value / steps.value.length) * 100)
})

const remainingDays = computed(() => {
  if (!isDurationLocked.value || !planStartDate.value || !selectedDuration.value) {
    return null
  }
  const start = new Date(planStartDate.value)
  const today = new Date()
  start.setHours(0, 0, 0, 0)
  today.setHours(0, 0, 0, 0)

  const endDate = new Date(start.getTime())
  endDate.setDate(start.getDate() + parseInt(selectedDuration.value, 10))

  const diffTime = endDate.getTime() - today.getTime()
  const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24))

  return diffDays >= 0 ? diffDays : 0
})

const triggerToast = () => {
  showToast.value = true
  setTimeout(() => {
    showToast.value = false
  }, 3000)
}

const savePlan = () => {
  const planData = {
    title: planTitle.value,
    duration: selectedDuration.value,
    steps: steps.value,
    startDate: planStartDate.value,
    durationLocked: isDurationLocked.value
  }
  localStorage.setItem('myPlan', JSON.stringify(planData))
  console.log('保存しました:', planData)
  triggerToast()
}

const loadPlan = () => {
  const saved = localStorage.getItem('myPlan')
  if (!saved) return
  try {
    const data = JSON.parse(saved)
    planTitle.value = data.title
    selectedDuration.value = data.duration
    steps.value = data.steps
    isEditingTitle.value = !data.title
    if (data.startDate) {
      planStartDate.value = new Date(data.startDate)
    }
    isDurationLocked.value = data.durationLocked ?? false
  } catch (e) {
    console.warn('読み込みエラー', e)
  }
}

onMounted(() => {
  loadPlan()
})
</script>

<style scoped>
@keyframes pop {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.4);
  }
  100% {
    transform: scale(1);
  }
}
.step-checkbox.animate {
  animation: pop 0.3s ease;
}

.my-plan {
  background: #f0f8ff; /* やわらかいブルー */
  border-radius: 12px;
  padding: 2rem;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  color: #445566;
}

.section-title {
  font-size: 1.6rem;
  font-weight: bold;
  color: #445566;
  margin-bottom: 1.5rem;
}

.card-section {
  background-color: #ffffff;
  border: 2px solid #dceefa; /* 薄い水色の枠線 */
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: 0 2px 8px rgba(145, 201, 247, 0.2);
  margin-bottom: 2rem;
  transition: box-shadow 0.3s ease;
}

.card-section:hover {
  box-shadow: 0 4px 12px rgba(145, 201, 247, 0.25);
}

.form-group {
  margin-bottom: 1.2rem;
  display: flex;
  flex-direction: column;
}
.form-group label {
  font-weight: 500;
  margin-bottom: 0.5rem;
}
input[type='text'],
select {
  background-color: #ffffff;
  border: 1px solid #cce3f5;
  border-radius: 6px;
  padding: 0.6rem 0.8rem;
  font-size: 1rem;
  color: #333;
}

.step-section {
  margin-top: 1rem;
}

.step-input {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
}
.step-input input {
  flex: 1;
  padding: 0.5rem;
  font-size: 1rem;
  border-radius: 6px;
  border: 1px solid #ccc;
}
.step-input button {
  padding: 0.5rem 1rem;
  background-color: #f8c8dc; /* ピンクアクセント */
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
}
.step-input button:disabled {
  background-color: #ddd;
  color: #aaa;
  cursor: not-allowed;
}

.step-cards {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  margin-top: 1rem;
}
.step-card {
  position: relative;
  background-color: #ffffff;
  border: 1px solid #ccc;
  border-radius: 8px;
  padding: 1rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
  transition: background-color 0.3s;
}
.step-card.completed {
  background-color: #eef6f9;
  border-color: #91c9f7;
}
.step-card-inner {
  cursor: pointer;
  padding-right: 2rem;
}
.step-text {
  display: inline-block;
  font-size: 1rem;
}
.step-checkbox {
  position: absolute;
  top: 1rem;
  right: 1rem;
  width: 20px;
  height: 20px;
  cursor: pointer;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.4);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}
.modal-content {
  background: white;
  border: 2px solid #dceefa;
  padding: 2rem;
  border-radius: 12px;
  max-width: 400px;
  width: 90%;
  box-shadow: 0 8px 16px rgba(145, 201, 247, 0.25);
  display: flex;
  flex-direction: column;
  gap: 1rem;
  text-align: center;
  color: #444;
}
.modal-content button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 1rem;
}
.modal-content button:first-of-type {
  background-color: #dc3545;
  color: white;
}
.modal-content button:last-of-type {
  background-color: #6c757d;
  color: white;
}

.template-list {
  list-style: none;
  padding: 0;
  margin: 1rem 0;
  text-align: left;
}

.template-item {
  padding: 0.75rem 1rem;
  border: 1px solid #dceefa;
  border-radius: 6px;
  margin-bottom: 0.5rem;
  cursor: pointer;
  transition: background-color 0.2s, border-color 0.2s;
}

.template-item:hover {
  background-color: #eaf6ff;
  border-color: #91c9f7;
}

.progress-bar-wrapper {
  margin-bottom: 1rem;
}
.progress-bar {
  background-color: #e0e0e0;
  height: 12px;
  border-radius: 8px;
  overflow: hidden;
}
.progress {
  height: 100%;
  background: linear-gradient(to right, #f8c8dc, #91c9f7); /* ピンク→ブルー */
  transition: width 0.3s ease;
}
.progress-text {
  font-size: 0.9rem;
  margin-top: 0.25rem;
  color: #555;
}

.goal-input-group {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}
.goal-input-group input[disabled] {
  background-color: #f1f1f1;
  color: #666;
  cursor: not-allowed;
}
.goal-input-group button {
  padding: 0.4rem 0.8rem;
  font-size: 0.9rem;
  border: none;
  border-radius: 6px;
  background-color: #f8c8dc;
  color: white;
  cursor: pointer;
  font-weight: bold;
  white-space: nowrap; /* ボタンのテキストが改行しないように */
}
.goal-input-group button:hover {
  background-color: #f4a9c4;
}

.duration-input-group {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}
.duration-input-group select[disabled] {
  background-color: #f1f1f1;
  color: #666;
  cursor: not-allowed;
}
.duration-input-group button {
  padding: 0.4rem 0.8rem;
  font-size: 0.9rem;
  border: none;
  border-radius: 6px;
  background-color: #f8c8dc;
  color: white;
  cursor: pointer;
  font-weight: bold;
  white-space: nowrap;
}
.duration-input-group button:hover {
  background-color: #f4a9c4;
}
.remaining-days {
  margin-top: 0.75rem;
  font-size: 0.95rem;
  font-weight: 500;
  color: #007bff;
}

.toast-notification {
  position: fixed;
  top: 20px;
  right: 20px;
  background-color: #28a745;
  color: white;
  padding: 0.75rem 1.25rem;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  z-index: 2000;
  font-weight: bold;
  transition: opacity 0.3s ease;
}
</style>


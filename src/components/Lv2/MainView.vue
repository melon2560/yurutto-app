<template>
  <div class="container">
    <MoodInput />
    <ActionGacha :level="2" />
    <MemoTracker />
    <MoodChart />
    <MoodCalendar />
    
  </div>
</template>

<script setup>
import MoodInput from './MoodInput.vue'
import ActionGacha from '../Lv1/ActionGacha.vue' // Lv1から再利用
import MemoTracker from './MemoTracker.vue' // 新しいメモ機能
import MoodChart from './MoodChart.vue' // Lv2専用のチャート
import MoodCalendar from './MoodCalendar.vue' // Lv2専用のカレンダー

</script>

<style scoped>
body {
  background-color: blue;
  margin: 0;
  font-family: sans-serif;
}
.container {
  max-width: 700px;
  margin: 0 auto;
  padding: 2rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);
}
</style>
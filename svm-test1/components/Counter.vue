
<script setup>
import { ref } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

function reset() {
  count.value = 0
}
</script>

<template>
  <div class="counter">
    <p class="counter-label">Vue component is working!</p>

    <p class="counter-value">{{ count }}</p>

    <div class="counter-buttons">
      <button @click="increment">Increment</button>
      <button @click="reset">Reset</button>
    </div>
  </div>
</template>

<style scoped>
.counter {
  padding: 24px;
  border: 2px solid #42b883;
  border-radius: 12px;
  text-align: center;
  max-width: 400px;
  margin: 30px auto;
}

.counter-label {
  font-size: 1.2rem;
  color: #42b883;
  font-weight: bold;
}

.counter-value {
  font-size: 3rem;
  font-weight: bold;
  margin: 16px 0;
}

.counter-buttons {
  display: flex;
  justify-content: center;
  gap: 12px;
}

.counter button {
  padding: 8px 16px;
  border: none;
  border-radius: 6px;
  background: #42b883;
  color: white;
  cursor: pointer;
}

.counter button:hover {
  background: #347a5a;
}
</style>

<!-- <script setup>
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <div>
    <a href="https://vite.dev" target="_blank">
      <img src="/vite.svg" class="logo" alt="Vite logo" />
    </a>
    <a href="https://vuejs.org/" target="_blank">
      <img src="./assets/vue.svg" class="logo vue" alt="Vue logo" />
    </a>
  </div>
  <HelloWorld msg="Vite + Vue" />
</template>

<style scoped>
.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: filter 300ms;
}
.logo:hover {
  filter: drop-shadow(0 0 2em #646cffaa);
}
.logo.vue:hover {
  filter: drop-shadow(0 0 2em #42b883aa);
}
</style> -->


<script setup>
import { ref, onMounted } from 'vue'

const questions = ref([])
const allQuestions = ref([]) // 保留完整題庫
const current = ref(0)
const selected = ref(null)
const showAnswer = ref(false)
const score = ref(0)
const mode = ref("practice") // "practice" or "exam"
const examAnswers = ref([])  // 存放模擬考的作答
const examSize = ref(10)     // 模擬考題數 (可變動)
const randomMode = ref(true) // 是否隨機出題

onMounted(async () => {
  const res = await fetch('./questions.json')
  const data = await res.json()
  allQuestions.value = data
  resetPractice()
})

function resetPractice() {
  mode.value = "practice"
  score.value = 0
  current.value = 0
  selected.value = null
  showAnswer.value = false

  let tempQuestions = [...allQuestions.value];
  if (randomMode.value) {
    tempQuestions.sort(() => Math.random() - 0.5);
  }
  questions.value = tempQuestions;
}

function startExam() {
  mode.value = "exam"
  score.value = 0
  current.value = 0
  selected.value = null
  examAnswers.value = []

  let pool = [...allQuestions.value];
  if (randomMode.value) {
    pool.sort(() => Math.random() - 0.5);
  }
  const size = Math.min(examSize.value, pool.length);
  questions.value = pool.slice(0, size);
}

function selectOption(letter) {
  selected.value = letter

  if (mode.value === "practice") {
    showAnswer.value = true
    if (letter === questions.value[current.value].correct_answer) {
      score.value++
    }
  } else {
    examAnswers.value.push({ qid: questions.value[current.value].id, answer: letter })
    if (letter === questions.value[current.value].correct_answer) {
      score.value++
    }
    nextQuestion()
  }
}

function nextQuestion() {
  selected.value = null
  showAnswer.value = false
  current.value++
}
</script>

<template>
  <div class="p-4 max-w-xl mx-auto text-gray-800">
    <!-- 模式切換 -->
    <div class="flex flex-wrap gap-2 mb-4 items-center">
      <button class="px-4 py-2 bg-green-500 text-white rounded" @click="resetPractice">
        練習模式
      </button>

      <input type="number " min="1" :max="allQuestions.length" v-model.number="examSize"
        class="w-20 p-2 border rounded text-center text-white" />
      <button class="px-4 py-2 bg-blue-500 text-white rounded" @click="startExam">
        模擬考模式
      </button>

      <!-- 隨機/順序切換 -->
      <label class="flex items-center gap-2 ml-2 text-white">
        <input type="checkbox" v-model="randomMode" @change="resetPractice" />
        隨機出題
      </label>
    </div>

    <!-- 練習模式 -->
    <div v-if="mode === 'practice' && current < questions.length">
      <div class="text-wrap ">
        <h2 class="break-all text-lg font-bold mb-2 text-left text-white">
          Q{{ questions[current]?.id }}. {{ questions[current]?.question }}
        </h2>
      </div>

      <div v-for="opt in questions[current]?.options" :key="opt.letter" class="mb-2">
        <button class="w-full p-2 border rounded text-left text-gray-300" :class="{
          'bg-green-200 text-black': showAnswer && opt.letter === questions[current].correct_answer,
          'bg-red-200 text-black': showAnswer && selected === opt.letter && opt.letter !== questions[current].correct_answer
        }" @click="selectOption(opt.letter)" :disabled="showAnswer">
          {{ opt.letter }}. {{ opt.text }}
        </button>
      </div>

      <div v-if="showAnswer" class="mt-4 ">
        <p class="text-sm text-gray-400 text-left mb-2">📘 解釋: {{ questions[current].explanation }}</p>
        <p class="text-sm text-green-700 text-left mb-2">👉 為什麼正確: {{ questions[current].why_correct }}</p>
        <p class="text-sm text-red-700 text-left">❌ 錯誤原因: 
          <div v-for="(reason, index) in questions[current].why_others_wrong" :key="index">
            <p>{{ reason }}</p>
          </div>
        </p>
        <!-- <p class="text-sm text-red-700"> 為什麼對: {{ questions[current].detailed_reasoning?.join('；') }}</p> -->

        <button class="mt-4 p-2 bg-blue-500 text-white rounded" @click="nextQuestion">
          下一題
        </button>
      </div>
    </div>

    <!-- 模擬考模式 -->
    <div v-if="mode === 'exam' && current < questions.length">
      <!-- 進度條 -->
      <div class="w-full bg-gray-200 rounded-full h-2.5 mb-4">
        <div class="bg-blue-500 h-2.5 rounded-full" :style="{ width: ((current + 1) / questions.length * 100) + '%' }">
        </div>
      </div>
      <p class="text-sm mb-2 text-gray-600">
        進度：{{ current + 1 }} / {{ questions.length }}
      </p>

      <h2 class="text-lg font-bold mb-2 text-left text-white">
        [模擬考] Q{{ current + 1 }}. {{ questions[current]?.question }}
      </h2>

      <div v-for="opt in questions[current]?.options" :key="opt.letter" class="mb-2">
        <button class="w-full p-2 border rounded text-left text-gray-300" @click="selectOption(opt.letter)">
          {{ opt.letter }}. {{ opt.text }}
        </button>
      </div>
    </div>

    <!-- 模擬考結束 -->
    <div v-if="mode === 'exam' && current >= questions.length" class="mt-6 text-center">
      <h3 class="text-xl font-bold text-white">🎉 模擬考結束</h3>
      <p class="text-white">你的分數：{{ score }} / {{ questions.length }}</p>

      <div v-for="(q, index) in questions" :key="q.id" class="mt-4 border-t pt-2 text-left">
        <h4 class="font-semibold text-left text-white">Q{{ index + 1 }}. {{ q.question }}</h4>
        <p class=" text-gray-400">你的答案: {{ examAnswers[index]?.answer || "未作答" }}</p>
        <p class=" text-gray-400">正確答案: {{ q.correct_answer }}</p>
        <p class="text-sm text-gray-200">📘 解釋: {{ q.explanation }}</p>
      </div>

      <button class="mt-6 px-4 py-2 bg-blue-600 text-white rounded" @click="startExam">
        🔄 再來一次模擬考
      </button>
    </div>

    <!-- 練習模式結束 -->
    <div v-if="mode === 'practice' && current >= questions.length" class="mt-6 text-center">
      <h3 class="text-xl font-bold">🎉 測驗結束</h3>
      <p>你的分數：{{ score }} / {{ questions.length }}</p>
      <button class="mt-4 px-4 py-2 bg-green-600 text-white rounded" @click="resetPractice">
        🔄 再練一次
      </button>
    </div>
  </div>
</template>

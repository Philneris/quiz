<template>
  <div class="min-h-screen bg-gradient-to-br from-blue-50 to-blue-100 flex items-center justify-center p-4">
    <div class="w-full max-w-2xl">
      <div v-if="!quizFinished" class="bg-white rounded-2xl shadow-2xl p-8">
        <div class="mb-6">
          <div class="flex justify-between items-center mb-4">
            <h1 class="text-3xl font-bold text-gray-800">Quiz Intelligent</h1>
            <span class="text-sm font-medium text-gray-600 bg-gray-100 px-4 py-2 rounded-full">
              Question {{ currentIndex + 1 }} sur {{ questions.length }}
            </span>
          </div>

          <div class="w-full bg-gray-200 rounded-full h-2.5 mb-6">
            <div
              class="bg-blue-600 h-2.5 rounded-full transition-all duration-300"
              :style="{ width: progressPercent + '%' }"
            ></div>
          </div>
        </div>

        <div class="mb-8">
          <h2 class="text-xl font-semibold text-gray-800 mb-6">
            {{ currentQuestion.question }}
          </h2>

          <input
            v-model="userAnswer"
            @keyup.enter="validateAnswer"
            :disabled="isAnswerValidated"
            type="text"
            placeholder="Votre réponse..."
            class="w-full px-4 py-3 border-2 border-gray-300 rounded-lg focus:border-blue-500 focus:outline-none text-lg disabled:bg-gray-100 disabled:cursor-not-allowed transition-colors"
          />
        </div>

        <div
          v-if="feedbackMessage"
          class="mb-6 p-4 rounded-lg transition-all duration-300"
          :class="isCorrect ? 'bg-green-100 border-2 border-green-500' : 'bg-red-100 border-2 border-red-500'"
        >
          <p class="font-semibold text-lg mb-2" :class="isCorrect ? 'text-green-800' : 'text-red-800'">
            {{ feedbackMessage }}
          </p>
          <p class="text-gray-700">
            <span class="font-medium">Bonne réponse :</span> {{ currentQuestion.reponse }}
          </p>
        </div>

        <div class="flex gap-4">
          <button
            @click="validateAnswer"
            :disabled="!userAnswer.trim() || isAnswerValidated"
            class="flex-1 bg-blue-600 text-white py-3 px-6 rounded-lg font-semibold hover:bg-blue-700 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors duration-200"
          >
            Valider
          </button>

          <button
            @click="nextQuestion"
            :disabled="!isAnswerValidated"
            class="flex-1 bg-gray-800 text-white py-3 px-6 rounded-lg font-semibold hover:bg-gray-900 disabled:bg-gray-300 disabled:cursor-not-allowed transition-colors duration-200"
          >
            Suivant
          </button>
        </div>
      </div>

      <div v-else class="bg-white rounded-2xl shadow-2xl p-8 text-center">
        <div class="mb-6">
          <svg class="w-24 h-24 mx-auto text-green-500 mb-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"></path>
          </svg>
          <h2 class="text-4xl font-bold text-gray-800 mb-2">Quiz Terminé !</h2>
        </div>

        <div class="bg-gradient-to-r from-blue-50 to-blue-100 rounded-xl p-6 mb-6">
          <p class="text-6xl font-bold text-blue-600 mb-2">{{ score }} / {{ questions.length }}</p>
          <p class="text-2xl font-semibold text-gray-700">{{ successPercent }}% de réussite</p>
        </div>

        <div class="mb-6">
          <p class="text-lg text-gray-600">{{ getEncouragementMessage() }}</p>
        </div>

        <button
          @click="restartQuiz"
          class="bg-blue-600 text-white py-3 px-8 rounded-lg font-semibold hover:bg-blue-700 transition-colors duration-200 text-lg"
        >
          Rejouer
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { compareTwoStrings } from 'string-similarity'
import questionsData from '../questions_propre.json'

// 🔀 Fonction pour mélanger les questions aléatoirement (algorithme de Fisher–Yates)
const shuffleArray = (array) => {
  const arr = [...array]
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[arr[i], arr[j]] = [arr[j], arr[i]]
  }
  return arr
}

// ✅ Mélange des questions avant le début du quiz
const questions = ref(shuffleArray(questionsData))

const currentIndex = ref(0)
const score = ref(0)
const userAnswer = ref('')
const isAnswerValidated = ref(false)
const feedbackMessage = ref('')
const isCorrect = ref(false)
const quizFinished = ref(false)

const currentQuestion = computed(() => questions.value[currentIndex.value])

const progressPercent = computed(() => ((currentIndex.value + 1) / questions.value.length) * 100)
const successPercent = computed(() => Math.round((score.value / questions.value.length) * 100))


// 🔤 Fonction pour normaliser et nettoyer la réponse
const normalizeString = (str) => {
  return str
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '') // Retire les accents
    .replace(/[^a-z0-9 ]/g, '') // Supprime la ponctuation
    .trim()
}

// 💡 Fonction pour comparer les réponses (tolérance aux fautes)
const validateAnswer = () => {
  if (!userAnswer.value.trim() || isAnswerValidated.value) return

  const userNormalized = normalizeString(userAnswer.value)
  const correctNormalized = normalizeString(currentQuestion.value.reponse)

  const similarity = compareTwoStrings(userNormalized, correctNormalized)

  // Vérifie si la réponse est assez longue pour être crédible
  const lengthRatio = userNormalized.length / correctNormalized.length

  // ✅ Conditions pour accepter la réponse :
  // - Similarité ≥ 0.75
  // - Longueur de la réponse utilisateur ≥ 60 % de la bonne réponse
  if (similarity >= 0.9) {
    isCorrect.value = true
    feedbackMessage.value = '✅ Bonne réponse !'
    score.value++
  } else {
    isCorrect.value = false
    feedbackMessage.value = '❌ Mauvaise réponse'
  }

  isAnswerValidated.value = true
}

const nextQuestion = () => {
  if (currentIndex.value < questions.value.length - 1) {
    currentIndex.value++
    userAnswer.value = ''
    isAnswerValidated.value = false
    feedbackMessage.value = ''
    isCorrect.value = false
  } else {
    quizFinished.value = true
  }
}

const restartQuiz = () => {
  currentIndex.value = 0
  score.value = 0
  userAnswer.value = ''
  isAnswerValidated.value = false
  feedbackMessage.value = ''
  isCorrect.value = false
  quizFinished.value = false
}

const getEncouragementMessage = () => {
  const percent = successPercent.value
  if (percent === 100) return 'Parfait ! Vous êtes un expert !'
  if (percent >= 80) return 'Excellent travail ! Continue comme ça !'
  if (percent >= 60) return 'Bon travail ! Tu peux encore t\'améliorer.'
  if (percent >= 40) return 'Pas mal ! Encore quelques efforts.'
  return 'Ne te décourage pas ! Réessaie pour t\'améliorer.'
}
</script>
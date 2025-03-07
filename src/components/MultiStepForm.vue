<template>
  <div class="card">
    <!-- Barra de progresso -->
    <div class="progress-bar-container">
      <div class="progress" :style="{ width: progress + 1 + '%' }"></div>
    </div>

    <!-- Options -->
    <transition name="fade" class="flex-container">
      <div v-show="showContent">
        <div class="flex-container" v-if="!finished">
          <h2>
            {{
              currentStep === 0
                ? 'Vamos começar com seu nome'
                : `${currentStep}/${formData.steps.length} ${formData.steps[currentStep - 1]?.question ?? ''}`
            }}
          </h2>

          <input
            v-if="currentStep === 0"
            type="text"
            v-model="userName"
            placeholder="Digite seu nome"
            class="name-input"
            @keydown.enter="onEnter"
          />

          <div class="form-options" v-else>
            <label
              v-for="(alternative, index) in formData.steps[currentStep - 1]?.alternatives"
              :key="index"
              class="form-option"
            >
              <input
                type="radio"
                v-model="formAnswers[currentStep - 1]"
                :id="`${currentStep}${index}`"
                :value="index"
              />
              {{ alternative }}
            </label>
          </div>
        </div>

        <!-- Finished Container -->
        <div v-else class="flex-container result-container">
          <h1>Você Finalizou!</h1>
          <h2>
            Arquétipos:
            {{
              getMostFrequentValues(formAnswers)
                .map((value) => formData.possibleResults[value])
                .join('-')
            }}
          </h2>
        </div>
      </div>
    </transition>

    <div class="separator" />

    <!-- Botões de Navegação -->
    <div class="navigation">
      <button @click="prevStep" :disabled="currentStep === 0 || !showContent">Voltar</button>

      <button
        :disabled="
          !showContent ||
          (currentStep === 0 && !userName) ||
          (currentStep > 0 && !Number.isInteger(formAnswers[currentStep - 1]))
        "
        v-if="currentStep <= formData.steps.length"
        @click="nextStep"
      >
        {{ currentStep !== formData.steps.length ? 'Próximo' : 'Finalizar' }}
      </button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from 'vue'
import formData from '@/data/formData.json'
import { sendEmail } from '@/api/email.ts'
import { useRouter } from 'vue-router'

const router = useRouter()

const currentStep = ref(0)
const formAnswers = ref([])
const finished = ref(false)
const showContent = ref(true)
const userName = ref('')
const canEnterInput = ref(true)

const nextStep = () => {
  showContent.value = false

  if (currentStep.value < formData.steps.length) {
    setTimeout(() => {
      currentStep.value++
      showContent.value = true
    }, 500)
  } else {
    setTimeout(() => {
      finished.value = true
      currentStep.value++
      showContent.value = true

      const archetypes = getMostFrequentValues(formAnswers.value)
      sendEmail(archetypes, userName.value)

      router.push({
        path: 'resultados',
        query: { archetypes },
      })
    }, 500)
  }
}

const prevStep = () => {
  showContent.value = false

  if (currentStep.value > 0) {
    setTimeout(() => {
      currentStep.value--
      finished.value = false
      showContent.value = true

      if (currentStep.value === 0) {
        canEnterInput.value = true
      }
    }, 500)
  }
}

const onEnter = () => {
  if (canEnterInput.value) {
    canEnterInput.value = false
    nextStep()
  }
}

const getMostFrequentValues = (arr: number[]): number[] => {
  const frequencyMap: Record<number, number> = arr.reduce(
    (acc, item) => {
      acc[item] = (acc[item] || 0) + 1
      return acc
    },
    {} as Record<number, number>,
  )

  return Object.keys(frequencyMap)
    .sort((a, b) => frequencyMap[+b] - frequencyMap[+a])
    .slice(0, 3)
    .map(Number)
}

const progress = computed(() => ((currentStep.value + 1) / (formData.steps.length + 1)) * 100)
</script>

<style scoped>
.card {
  display: flex;
  flex-direction: column;
  gap: var(--between-gap);
  background: var(--surface-container-high);
  color: var(--on-surface);
  max-height: 80vh;
  overflow: auto;
  padding: 2.5rem 3rem;
  border-radius: 5px;
  box-shadow: 0 3px 7px var(--shadow);
  border: 1px solid var(--surface-container-highest);
  text-align: center;
}

@media (max-width: 1024px) {
  .card {
    min-height: auto;
    max-height: none;
  }
}

.navigation {
  display: flex;
  justify-content: space-between;
}

.progress-bar-container {
  width: 100%;
  border-radius: 5px;
  height: var(--progress-bar-height);
  background-color: var(--background);
  box-shadow: 0 2px 5px var(--shadow-variant);
}

.progress {
  background-color: var(--primary);
  height: var(--progress-bar-height);
  transition: width 0.3s ease-in-out;
  border-radius: 5px;
}

.form-options {
  display: flex;
  flex-direction: column;
  align-items: baseline;
  gap: 1rem;
}

.form-option {
  text-align: left;
}

.form-option input {
  margin-right: 5px;
}

.flex-container {
  display: flex;
  flex-grow: 1;
  flex-direction: column;
  gap: var(--between-gap);
}

.result-container {
  align-items: center;
  justify-content: center;
}

.name-input {
  padding: 0.7rem;
  font-size: 1rem;
  border: 1px solid var(--surface-container-highest);
  border-radius: 20px;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to /* .fade-leave-active no Vue 2.1.8+ */ {
  opacity: 0;
}
</style>

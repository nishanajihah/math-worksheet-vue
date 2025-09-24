<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const API_URL_SCORES = 'https://math-worksheet-backend.vercel.app/api/scores'
const API_URL_QUESTIONS = 'https://math-worksheet-backend.vercel.app/api/questions'

// ================================
// REACTIVE STATE
// ================================
// Core application data
const questions = ref([])
const userAnswers = ref({})
const userName = ref('')
const score = ref(null)
const message = ref('')
const highScores = ref([])

// Application states
const isLoading = ref(false)
const isSubmitting = ref(false)
const dataLoaded = ref(false)
const backendAvailable = ref(true)
const criticalError = ref(false)

// Security and activation
const hasRealUser = ref(false)
const quizStarted = ref(false)
const honeypot = ref('')

// Rate limiting
let lastApiCall = 0

// ================================
// BOT DETECTION
// ================================
const isBot = () => {
  const ua = navigator.userAgent.toLowerCase()

  // Debug logging
  console.log('Bot check - User Agent:', navigator.userAgent)
  console.log('Bot check - webdriver:', navigator.webdriver)
  console.log('Bot check - callPhantom:', window.callPhantom)
  console.log('Bot check - _phantom:', window._phantom)

  if (ua.includes('bot')) {
    console.log('Bot detected: User agent contains "bot"')
    return true
  }
  if (ua.includes('crawler')) {
    console.log('Bot detected: User agent contains "crawler"')
    return true
  }
  if (ua.includes('spider')) {
    console.log('Bot detected: User agent contains "spider"')
    return true
  }
  if (ua.includes('scraper')) {
    console.log('Bot detected: User agent contains "scraper"')
    return true
  }
  if (ua.includes('headless')) {
    console.log('Bot detected: User agent contains "headless"')
    return true
  }
  if (navigator.webdriver === true) {
    console.log('Bot detected: webdriver is active')
    return true
  }
  if (window.callPhantom) {
    console.log('Bot detected: callPhantom detected')
    return true
  }
  if (window._phantom) {
    console.log('Bot detected: _phantom detected')
    return true
  }

  console.log('Bot check passed - legitimate user detected')
  return false
}

// ================================
// APP ACTIVATION
// ================================
const activateApp = () => {
  console.log('Start Quiz button clicked!')

  // Security checks
  if (isBot()) {
    console.log('App activation blocked - bot detected')
    return
  }

  if (honeypot.value) {
    console.log('App activation blocked - honeypot filled')
    return
  }

  // Activate application
  hasRealUser.value = true
  quizStarted.value = true
  console.log('Quiz started - loading data')

  // Load initial data
  loadInitialData()
}

// ================================
// DATA LOADING
// ================================
const loadInitialData = async () => {
  // Prevent duplicate loading
  if (dataLoaded.value || isLoading.value) return

  // STRICT: Must have clicked Start Quiz button
  if (!quizStarted.value) {
    console.log('API call BLOCKED - Start Quiz not clicked yet')
    return
  }

  // Security checks
  if (isBot()) {
    console.log('API call blocked - bot detected')
    return
  }

  if (!hasRealUser.value) {
    console.log('API call skipped - no user interaction detected')
    return
  }

  // Rate limiting
  const now = Date.now()
  if (now - lastApiCall < 10000) {
    console.log('API call rate limited - too soon after previous call')
    return
  }
  lastApiCall = now

  isLoading.value = true
  console.log('Starting to load questions and scores...')

  try {
    // Load questions and scores in parallel
    const [questionsResponse, scoresResponse] = await Promise.allSettled([
      axios.get(API_URL_QUESTIONS, { timeout: 10000 }),
      axios.get(API_URL_SCORES, { timeout: 10000 })
    ])

    console.log('API responses received:', {
      questionsStatus: questionsResponse.status,
      scoresStatus: scoresResponse.status
    })

    // Handle questions response
    if (questionsResponse.status === 'fulfilled') {
      questions.value = questionsResponse.value.data
      backendAvailable.value = true
      console.log('Questions loaded successfully:', questions.value.length, 'questions')
    } else {
      console.error('Failed to load questions:', questionsResponse.reason)
      backendAvailable.value = false
      criticalError.value = true
      message.value = 'Backend service is unavailable. Please refresh the page or try again later.'
    }

    // Handle scores response (optional)
    if (scoresResponse.status === 'fulfilled') {
      highScores.value = scoresResponse.value.data
      console.log('High scores loaded successfully:', highScores.value.length, 'entries')
    } else {
      console.log('High scores not available yet')
      highScores.value = []
    }

    dataLoaded.value = true

  } catch (error) {
    console.error('Critical error loading data:', error)
    backendAvailable.value = false
    criticalError.value = true
    message.value = 'Service temporarily unavailable. Please refresh the page to try again.'
  } finally {
    isLoading.value = false
  }
}

// ================================
// SCORE CALCULATION
// ================================
const calculateScore = async () => {
  // Prevent duplicate submissions
  if (isSubmitting.value) return

  // STRICT: Must have started quiz first
  if (!quizStarted.value) {
    console.log('Submission BLOCKED - Start Quiz not clicked yet')
    return
  }

  // Security checks
  if (isBot()) {
    console.log('Submission blocked - bot detected')
    return
  }

  if (honeypot.value) {
    console.log('Submission blocked - honeypot filled')
    return
  }

  if (!hasRealUser.value) {
    console.log('Submission blocked - no verified user interaction')
    return
  }

  // Backend availability check
  if (!backendAvailable.value) {
    message.value = 'Service is currently unavailable. Please refresh the page and try again.'
    return
  }

  // Form validation
  if (!userName.value?.trim()) {
    message.value = 'Please enter your name before submitting.'
    return
  }

  const trimmedName = userName.value.trim()
  if (trimmedName.length < 2 || trimmedName.length > 50) {
    message.value = 'Name must be between 2 and 50 characters.'
    return
  }

  const answeredCount = Object.keys(userAnswers.value).length
  if (answeredCount < questions.value.length) {
    message.value = `Please answer all ${questions.value.length} questions. You've answered ${answeredCount}.`
    return
  }

  isSubmitting.value = true

  try {
    const response = await axios.post(API_URL_SCORES, {
      name: trimmedName,
      userAnswers: userAnswers.value
    }, { timeout: 15000 })

    score.value = response.data.score
    message.value = response.data.message

    // Update high scores from response
    if (response.data.highScores) {
      highScores.value = response.data.highScores
    }

    // Reset form
    userAnswers.value = {}
    userName.value = ''

    console.log('Score submitted successfully')

  } catch (error) {
    console.error('Failed to submit score:', error)

    // Handle network/server errors
    if (error.code === 'ECONNABORTED' || error.response?.status >= 500) {
      backendAvailable.value = false
      criticalError.value = true
      message.value = 'Backend service is unavailable. Please refresh the page to try again.'
    } else {
      message.value = 'Failed to submit score. Please try again.'
    }
  } finally {
    isSubmitting.value = false
  }
}

// ================================
// UTILITY FUNCTIONS
// ================================
const resetWorksheet = () => {
  userName.value = ''
  score.value = null
  message.value = ''
  userAnswers.value = {}
}

const refreshHighScores = async () => {
  // STRICT: Must have started quiz first
  if (!quizStarted.value) {
    console.log('High scores refresh BLOCKED - Start Quiz not clicked yet')
    return
  }

  // Security checks
  if (isBot()) {
    console.log('High scores refresh blocked - bot detected')
    return
  }

  if (!hasRealUser.value) {
    console.log('High scores refresh blocked - no verified user interaction')
    return
  }

  if (!backendAvailable.value) {
    message.value = 'Service is currently unavailable. Please refresh the page to try again.'
    return
  }

  try {
    const response = await axios.get(API_URL_SCORES, { timeout: 10000 })
    highScores.value = response.data
    console.log('High scores refreshed')
  } catch (error) {
    console.error('Failed to refresh high scores:', error)

    if (error.code === 'ECONNABORTED' || error.response?.status >= 500) {
      backendAvailable.value = false
      message.value = 'Backend service is unavailable. Please refresh the page to try again.'
    } else {
      message.value = 'Failed to refresh high scores.'
    }
  }
}

const reloadPage = () => {
  window.location.reload()
}

// ================================
// LIFECYCLE
// ================================
onMounted(() => {
  console.log('App mounted - waiting for manual activation...')
  console.log('App in passive mode - manual activation required')
})
</script>

<template>
  <div class="glassmorphic-app">
    <!-- Navigation -->
    <nav class="glass-nav">
      <div class="nav-container">
        <div class="unified-controls-container">
          <div class="controls-left">
            <div class="control-buttons">
              <button
                @click="calculateScore"
                class="glass-btn primary-action-btn"
                :disabled="isSubmitting || isLoading || !backendAvailable"
              >
                <span class="btn-text">
                  {{ isSubmitting ? 'Submitting...' : !backendAvailable ? 'Service Unavailable' : 'Submit Answers' }}
                </span>
              </button>
              <button
                @click="resetWorksheet"
                class="glass-btn secondary-action-btn"
                :disabled="isSubmitting"
              >
                <span class="btn-text">Reset All</span>
              </button>
            </div>
          </div>
          <div class="controls-right">
            <div class="name-input-row">
              <label for="userName" class="input-label">Your Name</label>
              <input
                id="userName"
                v-model="userName"
                type="text"
                class="glass-input"
                placeholder="Enter your name..."
                maxlength="50"

              />

              <!-- Honeypot trap for bots (hidden from users) -->
              <input
                v-model="honeypot"
                type="text"
                style="position: absolute; left: -9999px; visibility: hidden;"
                tabindex="-1"
                autocomplete="off"
                aria-hidden="true"
              />
            </div>
          </div>
        </div>
      </div>
    </nav>

    <!-- Notifications -->
    <div v-if="message" class="notification-overlay" @click="message = ''">
      <div class="notification-dialog" @click.stop>
        <div class="notification-header">
          <div class="notification-title">
            {{ score !== null && !message.includes('Failed') ? 'Success!' : 'Attention!' }}
          </div>
          <button @click="message = ''" class="close-btn">×</button>
        </div>
        <div class="notification-content">
          <div class="notification-message">{{ message }}</div>
        </div>
        <div class="notification-footer">
          <button v-if="criticalError" @click="reloadPage" class="ok-btn refresh-page-btn">
            Refresh Page
          </button>
          <button v-else @click="message = ''" class="ok-btn">OK</button>
        </div>
      </div>
    </div>

    <!-- Main Content -->
    <main class="main-grid">
      <!-- Questions Panel -->
      <section class="glass-panel questions-panel">
        <div class="panel-header">
          <h2 class="panel-title">Rounding off to the nearest 10</h2>
          <p class="panel-subtitle">Circle the correct answer</p>
          <div class="progress-indicator">
            <span class="progress-text">
              Progress: {{ Object.keys(userAnswers).length }}/{{ questions.length }}
            </span>
            <div class="progress-bar">
              <div
                class="progress-fill"
                :style="{
                  width: questions.length ?
                    (Object.keys(userAnswers).length / questions.length) * 100 + '%' : '0%'
                }"
              ></div>
            </div>
          </div>
        </div>

        <!-- Waiting for Start Quiz Button Click -->
        <div v-if="!quizStarted && !isLoading" class="interaction-prompt">
          <div class="prompt-text">Welcome! Click to start your math challenge</div>
          <button @click="activateApp" class="glass-btn start-btn">Start Quiz</button>
        </div>

        <!-- Loading State -->
        <div v-else-if="isLoading" class="loading-container">
          <div class="loading-text">Loading math questions...</div>
        </div>

        <!-- Questions -->
        <div v-else-if="questions.length > 0" class="questions-container">
          <div
            v-for="(questionObj, index) in questions"
            :key="questionObj.id"
            class="glass-card question-card"
            :class="{ answered: userAnswers[questionObj.id] }"
          >
            <div class="question-number">{{ index + 1 }}</div>
            <div class="question-prompt">
              Round off {{ questionObj.question }} to the nearest 10
            </div>
            <div class="answer-grid">
              <button
                v-for="(choice, choiceIndex) in questionObj.choices"
                :key="choiceIndex"
                @click="userAnswers[questionObj.id] = choice"
                class="choice-btn"
                :class="{ selected: userAnswers[questionObj.id] === choice }"
              >
                <div class="choice-indicator">
                  <div class="choice-letter">{{ String.fromCharCode(65 + choiceIndex) }}</div>
                  <div class="choice-circle">
                    <div
                      class="circle-dot"
                      :class="{ active: userAnswers[questionObj.id] === choice }"
                    ></div>
                  </div>
                </div>
                <div class="choice-text">{{ choice }}</div>
                <div class="choice-glow"></div>
              </button>
            </div>
          </div>
        </div>

        <!-- Error State -->
        <div v-else-if="quizStarted && !isLoading && (questions.length === 0 || criticalError)" class="error-container">
          <div class="error-text">Failed to load questions</div>
          <button @click="reloadPage" class="glass-btn retry-btn refresh-page-btn">Refresh Page</button>
        </div>
      </section>

      <!-- Sidebar -->
      <aside class="sidebar-panel">
        <div class="glass-panel leaderboard-section">
          <div class="section-header">
            <h3 class="section-title">Highscores Top 10</h3>
            <button
              v-if="backendAvailable && highScores.length > 0"
              @click="refreshHighScores"
              class="glass-btn refresh-btn"
              :disabled="isLoading"
            >
              ↻ Refresh
            </button>
          </div>

          <!-- Waiting for Quiz Start -->
          <div v-if="!quizStarted" class="empty-leaderboard">
            <div class="empty-text">Leaderboard</div>
            <div class="empty-subtext">Start the quiz to see high scores!</div>
          </div>

          <!-- Empty or Loading Leaderboard -->
          <div v-else-if="highScores.length === 0" class="empty-leaderboard">
            <div class="empty-text">No high scores yet!</div>
            <div class="empty-subtext">Be the first to complete the challenge</div>
          </div>

          <!-- High Scores List -->
          <div v-else class="champions-list">
            <div
              v-for="(hs, index) in highScores.slice(0, 10)"
              :key="index"
              class="glass-card champion-item"
              :class="`rank-${index + 1}`"
            >
              <div class="champion-rank">
                <div v-if="index < 3" class="rank-medal">
                  {{ index === 0 ? '🥇' : index === 1 ? '🥈' : '🥉' }}
                </div>
                <div v-else class="rank-badge">{{ index + 1 }}</div>
              </div>
              <div class="champion-info">
                <div class="champion-name">{{ hs.name }}</div>
                <div class="champion-score">{{ hs.score }}/12</div>
              </div>
              <div class="champion-percentage">{{ hs.score }}/12</div>
            </div>
          </div>
        </div>
      </aside>
    </main>

    <!-- Footer -->
    <footer class="glass-footer">
      <div class="footer-content">
        <div class="copyright-box">
          <span class="copyright-text">copyright: mathinenglish.com</span>
        </div>
      </div>
      <div class="footer-waves">
        <div class="wave wave-1"></div>
        <div class="wave wave-2"></div>
      </div>
    </footer>
  </div>
</template>

<style>
@import './css/global.css';
@import './css/variables.css';
@import './css/navigation.css';
@import './css/layout.css';
@import './css/notifications.css';
@import './css/questions.css';
@import './css/choices.css';
@import './css/leaderboard.css';
@import './css/footer.css';
@import './css/responsive.css';


</style>

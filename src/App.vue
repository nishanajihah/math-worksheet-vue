<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

// ================================
// CONFIGURATION
// ================================
const API_URL_SCORES = 'https://math-worksheet-backend.vercel.app/api/scores'
const API_URL_QUESTIONS = 'https://math-worksheet-backend.vercel.app/api/questions'
const IS_DEVELOPMENT = import.meta.env.DEV
const RATE_LIMIT_MS = 10000 // 10 seconds between API calls

// ================================
// FALLBACK DATA (OFFLINE MODE)
// ================================
const FALLBACK_QUESTIONS = [
  { id: 'q1', question: '17', correctAnswer: '20', choices: ['10', '20', '17'] },
  { id: 'q2', question: '75', correctAnswer: '80', choices: ['70', '80', '75'] },
  { id: 'q3', question: '64', correctAnswer: '60', choices: ['64', '70', '60'] },
  { id: 'q4', question: '98', correctAnswer: '100', choices: ['80', '100', '98'] },
  { id: 'q5', question: '94', correctAnswer: '90', choices: ['100', '94', '90'] },
  { id: 'q6', question: '445', correctAnswer: '450', choices: ['450', '440', '500'] },
  { id: 'q7', question: '45', correctAnswer: '50', choices: ['50', '45', '40'] },
  { id: 'q8', question: '19', correctAnswer: '20', choices: ['20', '10', '19'] },
  { id: 'q9', question: '0', correctAnswer: '0', choices: ['10', '1', '0'] },
  { id: 'q10', question: '199', correctAnswer: '200', choices: ['190', '100', '200'] },
  { id: 'q11', question: '165', correctAnswer: '170', choices: ['160', '170', '150'] },
  { id: 'q12', question: '999', correctAnswer: '1000', choices: ['990', '1000', '909'] }
]

const FALLBACK_HIGH_SCORES = [
  { name: "Alice", score: 5, date: "2024-01-15" },
  { name: "Bob", score: 4, date: "2024-01-14" },
  { name: "Charlie", score: 3, date: "2024-01-13" }
]

// ================================
// REACTIVE STATE - CORE DATA
// ================================
const questions = ref([])
const userAnswers = ref({})
const userName = ref('')
const score = ref(null)
const message = ref('')
const highScores = ref([])

// ================================
// REACTIVE STATE - APPLICATION STATUS
// ================================
const isLoading = ref(false)
const isSubmitting = ref(false)
const dataLoaded = ref(false)
const backendAvailable = ref(true)
const criticalError = ref(false)

// ================================
// REACTIVE STATE - SECURITY & ACTIVATION
// ================================
const hasRealUser = ref(false)
const quizStarted = ref(false)
const honeypot = ref('') // Bot detection honeypot

// ================================
// UTILITY - LOGGING
// ================================
const debugLog = (...args) => {
  if (IS_DEVELOPMENT) {
    console.log(...args)
  }
}

const errorLog = (...args) => {
  if (IS_DEVELOPMENT) {
    console.error(...args)
  }
}

// ================================
// UTILITY - RATE LIMITING
// ================================
let lastApiCall = 0

const isRateLimited = () => {
  const now = Date.now()
  if (now - lastApiCall < RATE_LIMIT_MS) {
    debugLog('API call rate limited - too soon after previous call')
    return true
  }
  lastApiCall = now
  return false
}

// ================================
// SECURITY - BOT DETECTION
// ================================
const isBot = () => {
  const ua = navigator.userAgent.toLowerCase()

  // Bot patterns to detect
  const botPatterns = ['bot', 'crawler', 'spider', 'scraper', 'headless']

  // Check user agent for bot patterns
  for (const pattern of botPatterns) {
    if (ua.includes(pattern)) {
      debugLog(`Bot detected: User agent contains "${pattern}"`)
      return true
    }
  }

  // Check for automation tools
  if (navigator.webdriver === true) {
    debugLog('Bot detected: webdriver is active')
    return true
  }

  if (window.callPhantom || window._phantom) {
    debugLog('Bot detected: PhantomJS detected')
    return true
  }

  debugLog('Bot check passed - legitimate user detected')
  return false
}

// ================================
// CORE FUNCTIONALITY - APP ACTIVATION
// ================================
const activateApp = () => {
  debugLog('Start Quiz button clicked!')

  // Security validation
  if (isBot()) {
    debugLog('App activation blocked - bot detected')
    return
  }

  if (honeypot.value) {
    debugLog('App activation blocked - honeypot trap triggered')
    return
  }

  // Activate quiz mode
  hasRealUser.value = true
  quizStarted.value = true
  debugLog('Quiz activated - loading initial data')

  // Load questions and scores
  loadInitialData()
}

// ================================
// CORE FUNCTIONALITY - DATA LOADING
// ================================
const loadInitialData = async () => {
  // Prevent duplicate API calls
  if (dataLoaded.value || isLoading.value) return

  // Security: Ensure quiz has been properly activated
  if (!quizStarted.value) {
    debugLog('API call blocked - quiz not started')
    return
  }

  if (isBot() || !hasRealUser.value) {
    debugLog('API call blocked - security check failed')
    return
  }

  // Rate limiting protection
  if (isRateLimited()) return

  isLoading.value = true
  debugLog('Loading questions and high scores from API...')

  try {
    // Attempt to load questions and scores in parallel
    const [questionsResponse, scoresResponse] = await Promise.allSettled([
      axios.get(API_URL_QUESTIONS, { timeout: 10000 }),
      axios.get(API_URL_SCORES, { timeout: 10000 })
    ])

    // Determine success status for both endpoints
    const questionsSuccess = questionsResponse.status === 'fulfilled'
    const scoresSuccess = scoresResponse.status === 'fulfilled'

    debugLog('API Response Status:', { questionsSuccess, scoresSuccess })

    // Backend is available only if BOTH endpoints respond successfully
    backendAvailable.value = questionsSuccess && scoresSuccess

    // Load questions (with fallback)
    if (questionsSuccess) {
      questions.value = questionsResponse.value.data
      debugLog(`Questions loaded: ${questions.value.length} items`)
    } else {
      errorLog('Questions API failed:', questionsResponse.reason)
      questions.value = FALLBACK_QUESTIONS
      message.value = 'Running in offline mode with sample questions.'
    }

    // Load high scores (with fallback)
    if (scoresSuccess) {
      highScores.value = scoresResponse.value.data
      debugLog(`High scores loaded: ${highScores.value.length} entries`)
    } else {
      debugLog('High scores unavailable, using fallback data')
      highScores.value = FALLBACK_HIGH_SCORES
    }

    dataLoaded.value = true

  } catch (error) {
    errorLog('Critical error during data loading:', error)
    backendAvailable.value = false
    criticalError.value = true
    message.value = 'Service temporarily unavailable. Please refresh the page to try again.'
  } finally {
    isLoading.value = false
  }
}

// ================================
// CORE FUNCTIONALITY - SCORE SUBMISSION
// ================================
const calculateScore = async () => {
  // Prevent duplicate submissions
  if (isSubmitting.value) return

  // Security validation
  if (!quizStarted.value || isBot() || honeypot.value || !hasRealUser.value) {
    debugLog('Score submission blocked - security check failed')
    return
  }

  // Service availability check
  if (!backendAvailable.value) {
    message.value = 'Service is currently unavailable. Please refresh the page and try again.'
    return
  }

  // Input validation
  const trimmedName = userName.value?.trim()
  if (!trimmedName) {
    message.value = 'Please enter your name before submitting.'
    return
  }

  if (trimmedName.length < 2 || trimmedName.length > 50) {
    message.value = 'Name must be between 2 and 50 characters.'
    return
  }

  // Completion validation
  const answeredCount = Object.keys(userAnswers.value).length
  if (answeredCount < questions.value.length) {
    message.value = `Please answer all ${questions.value.length} questions. You've answered ${answeredCount}.`
    return
  }

  isSubmitting.value = true
  debugLog('Submitting quiz answers for scoring...')

  try {
    const response = await axios.post(API_URL_SCORES, {
      name: trimmedName,
      userAnswers: userAnswers.value
    }, { timeout: 15000 })

    // Process successful response
    score.value = response.data.score
    message.value = response.data.message

    // Update leaderboard if provided
    if (response.data.highScores) {
      highScores.value = response.data.highScores
    }

    // Reset quiz for new attempt
    userAnswers.value = {}
    userName.value = ''

    debugLog('Quiz completed and score submitted successfully')

  } catch (error) {
    errorLog('Score submission failed:', error)

    // Handle different error types
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
  // Security and state validation
  if (!quizStarted.value || isBot() || !hasRealUser.value || !backendAvailable.value) {
    debugLog('High scores refresh blocked - validation failed')
    return
  }

  try {
    const response = await axios.get(API_URL_SCORES, { timeout: 10000 })
    highScores.value = response.data
    debugLog('High scores refreshed successfully')
  } catch (error) {
    errorLog('Failed to refresh high scores:', error)

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
// COMPONENT LIFECYCLE
// ================================
onMounted(() => {
  debugLog('Math Worksheet App initialized - waiting for user activation')
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

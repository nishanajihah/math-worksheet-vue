<script setup>
import { ref, onMounted } from 'vue';
import axios from 'axios';

// Backend URLs
const API_URL_SCORES = 'https://math-worksheet-backend.vercel.app/api/scores';
const API_URL_QUESTIONS = 'https://math-worksheet-backend.vercel.app/api/questions';

// Core data
const questions = ref([]);
const userAnswers = ref({});
const userName = ref('');
const score = ref(null);
const message = ref('');
const highScores = ref([]);

// Simple loading states (no complex caching)
const isLoading = ref(false);
const isSubmitting = ref(false);
const dataLoaded = ref(false);
const backendAvailable = ref(true);
const criticalError = ref(false);

// SOLUTION 1: Single API call on mount (no interaction detection)
const loadInitialData = async () => {
  if (dataLoaded.value || isLoading.value) return;

  isLoading.value = true;

  try {
    // Load questions and scores in parallel (only once!)
    const [questionsResponse, scoresResponse] = await Promise.allSettled([
      axios.get(API_URL_QUESTIONS, { timeout: 10000 }),
      axios.get(API_URL_SCORES, { timeout: 10000 })
    ]);

    // Handle questions
    if (questionsResponse.status === 'fulfilled') {
      questions.value = questionsResponse.value.data;
      backendAvailable.value = true;
      console.log('✅ Questions loaded successfully');
    } else {
      console.error('❌ Failed to load questions:', questionsResponse.reason);
      backendAvailable.value = false;
      criticalError.value = true;
      message.value = 'Backend service is unavailable. Please refresh the page or try again later.';
    }

    // Handle scores (optional, don't fail if this doesn't work)
    if (scoresResponse.status === 'fulfilled') {
      highScores.value = scoresResponse.value.data;
      console.log('✅ High scores loaded successfully');
    } else {
      console.log('ℹ️ High scores not available yet');
      highScores.value = [];
      // Don't mark backend as unavailable just for scores failing
    }

    dataLoaded.value = true;

  } catch (error) {
    console.error('❌ Critical error loading data:', error);
    backendAvailable.value = false;
    criticalError.value = true;
    message.value = 'Service temporarily unavailable. Please refresh the page to try again.';
  } finally {
    isLoading.value = false;
  }
};

// SOLUTION 2: Only submit when user actually submits
const calculateScore = async () => {
  if (isSubmitting.value) return;

  // Check if backend is available
  if (!backendAvailable.value) {
    message.value = 'Service is currently unavailable. Please refresh the page and try again.';
    return;
  }

  // Validation
  if (!userName.value?.trim()) {
    message.value = 'Please enter your name before submitting.';
    return;
  }

  const trimmedName = userName.value.trim();
  if (trimmedName.length < 2 || trimmedName.length > 50) {
    message.value = 'Name must be between 2 and 50 characters.';
    return;
  }

  const answeredCount = Object.keys(userAnswers.value).length;
  if (answeredCount < questions.value.length) {
    message.value = `Please answer all ${questions.value.length} questions. You've answered ${answeredCount}.`;
    return;
  }

  isSubmitting.value = true;

  try {
    const response = await axios.post(API_URL_SCORES, {
      name: trimmedName,
      userAnswers: userAnswers.value
    }, { timeout: 15000 });

    score.value = response.data.score;
    message.value = response.data.message;

    // SOLUTION 3: Update high scores from response (no extra API call!)
    if (response.data.highScores) {
      highScores.value = response.data.highScores;
    }

    // Reset form
    userAnswers.value = {};
    userName.value = '';

    console.log('✅ Score submitted successfully');

  } catch (error) {
    console.error('❌ Failed to submit score:', error);

    // If it's a network or server error, mark backend as unavailable
    if (error.code === 'ECONNABORTED' || error.response?.status >= 500) {
      backendAvailable.value = false;
      criticalError.value = true;
      message.value = 'Backend service is unavailable. Please refresh the page to try again.';
    } else {
      message.value = 'Failed to submit score. Please try again.';
    }
  } finally {
    isSubmitting.value = false;
  }
};

// SOLUTION 4: Simple reset (no API calls)
const resetWorksheet = () => {
  userName.value = '';
  score.value = null;
  message.value = '';
  userAnswers.value = {};
};

// SOLUTION 5: Manual refresh only when user clicks (rare)
const refreshHighScores = async () => {
  if (!backendAvailable.value) {
    message.value = 'Service is currently unavailable. Please refresh the page to try again.';
    return;
  }

  try {
    const response = await axios.get(API_URL_SCORES, { timeout: 10000 });
    highScores.value = response.data;
    console.log('✅ High scores refreshed');
  } catch (error) {
    console.error('❌ Failed to refresh high scores:', error);

    // If it's a server error, mark backend as unavailable
    if (error.code === 'ECONNABORTED' || error.response?.status >= 500) {
      backendAvailable.value = false;
      message.value = 'Backend service is unavailable. Please refresh the page to try again.';
    } else {
      message.value = 'Failed to refresh high scores.';
    }
  }
};

// Reload page when backend is unavailable
const reloadPage = () => {
  window.location.reload();
};

// SOLUTION 6: Load data immediately on mount (no user interaction detection)
onMounted(() => {
  console.log('🚀 App mounted, loading data...');
  loadInitialData();
});
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

        <!-- Loading State -->
        <div v-if="isLoading" class="loading-container">
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
        <div v-else class="error-container">
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

          <!-- Empty or Loading Leaderboard -->
          <div v-if="highScores.length === 0" class="empty-leaderboard">
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

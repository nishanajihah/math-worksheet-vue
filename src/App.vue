<script setup>
// import ref to create reactive variable and onmounted to run code
// import axios to make API request to our backend
import { ref, onMounted } from 'vue';
import axios from 'axios';

// URL of our backend API
const API_URL_SCORES = 'https://math-worksheet-backend.vercel.app/api/scores';
const API_URL_QUESTIONS = 'https://math-worksheet-backend.vercel.app/api/questions';

// Reactive state to hold the user's data and app status.
const questions = ref([]);
const userAnswers = ref({});
const userName = ref('');
const score = ref(null);
const message = ref('');
const highScores = ref([]);

// Control API call states
const questionsLoaded = ref(false);
const highScoresLoaded = ref(false);
const isUserInteracting = ref(false);
const isLoadingQuestions = ref(false);
const isLoadingHighScores = ref(false);
const isSubmittingScore = ref(false);

// Cache for responses
const questionsCache = ref(null);
const highScoresCache = ref(null);
const cacheTimestamp = ref(null);
const CACHE_DURATION = 5 * 60 * 1000; // 5 minutes

// Check if cache is still valid
const isCacheValid = (timestamp) => {
  return timestamp && (Date.now() - timestamp) < CACHE_DURATION;
};

// Fetch the questions only when user interacts
const fetchQuestions = async () => {
  if (questionsLoaded.value || isLoadingQuestions.value) return; // Don't fetch if already loaded or loading

  // Check cache first
  if (questionsCache.value && isCacheValid(cacheTimestamp.value)) {
    questions.value = questionsCache.value;
    questionsLoaded.value = true;
    return;
  }

  isLoadingQuestions.value = true;
  try {
    // Add timeout to prevent hanging requests (10 seconds)
    const response = await Promise.race([
      axios.get(API_URL_QUESTIONS),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Request timeout')), 10000)
      )
    ]);

    questions.value = response.data;
    questionsCache.value = response.data; // Cache the response
    cacheTimestamp.value = Date.now();
    questionsLoaded.value = true;
  } catch (error) {
    console.error('Failed to fetch questions:', error);
    if (error.message === 'Request timeout') {
      message.value = "Request timed out. Please check your connection and try again.";
    } else {
      message.value = "Failed to load questions. Click to retry.";
    }
  } finally {
    isLoadingQuestions.value = false;
  }
};

// Fetch latest high scores from backend only when needed
const fetchHighScores = async () => {
  if (highScoresLoaded.value || isLoadingHighScores.value) return; // Don't fetch if already loaded or loading

  // Check cache first (shorter cache for leaderboard - 2 minutes)
  if (highScoresCache.value && cacheTimestamp.value &&
      (Date.now() - cacheTimestamp.value) < (2 * 60 * 1000)) {
    highScores.value = highScoresCache.value;
    highScoresLoaded.value = true;
    return;
  }

  isLoadingHighScores.value = true;
  try {
    // Add timeout to prevent hanging requests (10 seconds)
    const response = await Promise.race([
      axios.get(API_URL_SCORES),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Request timeout')), 10000)
      )
    ]);

    highScores.value = response.data;
    highScoresCache.value = response.data; // Cache the response
    highScoresLoaded.value = true;
  } catch (error) {
    console.error('Failed to fetch high scores:', error);
    if (error.message === 'Request timeout') {
      message.value = "Request timed out. Please check your connection and try again.";
    } else {
      message.value = "Failed to load high scores. Click to retry.";
    }
  } finally {
    isLoadingHighScores.value = false;
  }
};

// Manual refresh high scores (force reload)
const refreshHighScores = async () => {
  // Clear cache to force fresh data
  highScoresCache.value = null;
  highScoresLoaded.value = false;
  await fetchHighScores();
};

// Detect user interaction and load data
const handleUserInteraction = async () => {
  if (isUserInteracting.value) return; // Already detected interaction

  isUserInteracting.value = true;

  // Load questions first (needed for worksheet)
  if (!questionsLoaded.value) {
    await fetchQuestions();
  }

  // Load high scores (less critical, can be delayed)
  setTimeout(async () => {
    if (!highScoresLoaded.value) {
      await fetchHighScores();
    }
  }, 1000); // Delay high scores by 1 second
};

// Calculate the score and submit it to the backend
const calculateScore = async () => {
  // Prevent double-clicks with loading state
  if (isSubmittingScore.value) return;

  // Validate user name
  if (!userName.value || userName.value.trim().length === 0) {
    message.value = "Please enter your name before submitting.";
    return;
  }

  // Validate name length and characters
  const trimmedName = userName.value.trim();
  if (trimmedName.length < 2) {
    message.value = "Name must be at least 2 characters long.";
    return;
  }

  if (trimmedName.length > 50) {
    message.value = "Name must be less than 50 characters.";
    return;
  }

  // Check if all questions are answered
  const totalQuestions = questions.value.length;
  const answeredQuestions = Object.keys(userAnswers.value).length;

  if (answeredQuestions < totalQuestions) {
    message.value = `Please complete all questions before submitting. You have answered ${answeredQuestions} out of ${totalQuestions} questions.`;
    return;
  }

  // Validate that all answers are valid
  const validAnswers = Object.values(userAnswers.value).every(answer =>
    answer !== null && answer !== undefined && answer !== ''
  );

  if (!validAnswers) {
    message.value = "Please ensure all answers are properly selected.";
    return;
  }

  isSubmittingScore.value = true;

  // Submit name and score to backend
  try {
    // Add timeout to prevent hanging requests (15 seconds for submission)
    const response = await Promise.race([
      axios.post(API_URL_SCORES, {
        name: trimmedName,
        userAnswers: userAnswers.value
      }),
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Request timeout')), 15000)
      )
    ]);

    score.value = response.data.score;
    message.value = response.data.message;

    // Clear high scores cache and force refresh after score submission
    highScoresCache.value = null;
    highScoresLoaded.value = false;
    await fetchHighScores();

    // Auto-reset the worksheet after successful submission
    userAnswers.value = {};

    // Scroll to highscores section after successful submission
    setTimeout(() => {
      const highscoreSection = document.querySelector('.leaderboard-section');
      if (highscoreSection) {
        highscoreSection.scrollIntoView({ behavior: 'smooth', block: 'center' });
      }
    }, 500);

  } catch (error) {
    console.error('Failed to submit score:', error);
    if (error.message === 'Request timeout') {
      message.value = "Submission timed out. Please check your connection and try again.";
    } else {
      message.value = "Failed to submit score. Please try again.";
    }
  } finally {
    isSubmittingScore.value = false;
  }
};

// Resets the worksheet to its initial state.
const resetWorksheet = () => {
  userName.value = '';
  score.value = null;
  message.value = '';
  userAnswers.value = {};
};

// Initialize app without automatic API calls
onMounted(() => {
  // Add event listeners for user interactions
  const interactionEvents = ['click', 'keydown', 'scroll', 'mousemove', 'touchstart'];

  const handleFirstInteraction = () => {
    handleUserInteraction();
    // Remove listeners after first interaction
    interactionEvents.forEach(event => {
      document.removeEventListener(event, handleFirstInteraction);
    });
  };

  // Listen for any user interaction
  interactionEvents.forEach(event => {
    document.addEventListener(event, handleFirstInteraction, { passive: true });
  });
});

</script>

<template>
  <div class="glassmorphic-app">
    <!-- Glass Navigation -->
    <nav class="glass-nav">
      <div class="nav-container">
        <div class="unified-controls-container">
          <div class="controls-left">
            <div class="control-buttons">
              <button @click="calculateScore" class="glass-btn primary-action-btn" :disabled="isSubmittingScore">
                <span class="btn-text">{{ isSubmittingScore ? 'Submitting...' : 'Submit Answers' }}</span>
              </button>
              <button @click="resetWorksheet" class="glass-btn secondary-action-btn" :disabled="isSubmittingScore">
                <span class="btn-text">Reset All</span>
              </button>
            </div>
          </div>
          <div class="controls-right">
            <div class="name-input-row">
              <label for="userName" class="input-label">Your Name</label>
              <input id="userName" v-model="userName" type="text" class="glass-input" placeholder="Enter your name..."
                maxlength="50" @focus="handleUserInteraction" />
            </div>
          </div>
        </div>
      </div>
    </nav>

    <!-- Success/Error Notifications -->
    <div v-if="message" class="notification-overlay" @click="message = ''">
      <div class="notification-dialog" @click.stop>
        <div class="notification-header">
          <div class="notification-title">{{ (score !== null && !message.includes('Failed')) ? 'Success!' : 'Attention!'
            }}</div>
          <button @click="message = ''" class="close-btn">×</button>
        </div>
        <div class="notification-content">
          <div class="notification-message">{{ message }}</div>
        </div>
        <div class="notification-footer">
          <button @click="message = ''" class="ok-btn">OK</button>
        </div>
      </div>
    </div>

    <!-- Main Content Grid -->
    <main class="main-grid">
      <!-- Questions Panel -->
      <section class="glass-panel questions-panel">
        <div class="panel-header">
          <h2 class="panel-title">
            Rounding off to the nearest 10
          </h2>
          <p class="panel-subtitle">Circle the correct answer</p>
          <div class="progress-indicator">
            <span class="progress-text">Progress: {{ Object.keys(userAnswers).length }}/{{ questions.length }}</span>
            <div class="progress-bar">
              <div class="progress-fill"
                :style="{ width: questions.length ? (Object.keys(userAnswers).length / questions.length) * 100 + '%' : '0%' }">
              </div>
            </div>
          </div>
        </div>

        <!-- Loading state for questions -->
        <div v-if="isLoadingQuestions" class="loading-container">
          <div class="loading-text">Loading math questions...</div>
        </div>

        <!-- No questions loaded yet -->
        <div v-else-if="!questionsLoaded && !isLoadingQuestions" class="load-prompt">
          <div class="load-text">Ready to start your math challenge?</div>
          <button @click="handleUserInteraction" class="glass-btn load-btn">Load Questions</button>
        </div>

        <div v-else class="questions-container">
          <div v-for="(questionObj, index) in questions" :key="questionObj.id" class="glass-card question-card"
            :class="{ answered: userAnswers[questionObj.id] }">
            <div class="question-number">{{ index + 1 }}</div>
            <div class="question-prompt">Round off {{ questionObj.question }} to the nearest 10</div>
            <div class="answer-grid">
              <button v-for="(choice, choiceIndex) in questionObj.choices" :key="choiceIndex"
                @click="userAnswers[questionObj.id] = choice" class="choice-btn"
                :class="{ selected: userAnswers[questionObj.id] === choice }">
                <div class="choice-indicator">
                  <div class="choice-letter">{{ String.fromCharCode(65 + choiceIndex) }}</div>
                  <div class="choice-circle">
                    <div class="circle-dot" :class="{ active: userAnswers[questionObj.id] === choice }"></div>
                  </div>
                </div>
                <div class="choice-text">{{ choice }}</div>
                <div class="choice-glow"></div>
              </button>
            </div>
          </div>
        </div>
      </section>

      <!-- Sidebar Panel -->
      <aside class="sidebar-panel">
        <!-- Leaderboard Section -->
        <div class="glass-panel leaderboard-section">
          <div class="section-header">
            <h3 class="section-title">Highscores Top 10</h3>
            <button v-if="highScoresLoaded" @click="refreshHighScores" class="glass-btn refresh-btn" :disabled="isLoadingHighScores">
              {{ isLoadingHighScores ? '⟳' : '↻' }} Refresh
            </button>
          </div>

          <!-- Loading high scores -->
          <div v-if="isLoadingHighScores" class="loading-container">
            <div class="loading-text">Loading leaderboard...</div>
          </div>

          <!-- No high scores loaded yet -->
          <div v-else-if="!highScoresLoaded && !isLoadingHighScores" class="load-prompt">
            <div class="load-text">View the leaderboard</div>
            <button @click="handleUserInteraction" class="glass-btn load-btn">Load Scores</button>
          </div>

          <!-- Empty leaderboard -->
          <div v-else-if="highScores.length === 0" class="empty-leaderboard">
            <div class="empty-text">No Highscore yet!</div>
            <div class="empty-subtext">Be the first to complete the challenge</div>
          </div>

          <div v-else class="champions-list">
            <div v-for="(hs, index) in highScores.slice(0, 5)" :key="index" class="glass-card champion-item"
              :class="`rank-${index + 1}`">
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

    <!-- Glassmorphic Footer -->
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

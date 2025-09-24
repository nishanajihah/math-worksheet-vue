<script setup>
// import ref to create reactive variable and onmounted to run code
// import axios to make API request to our backend
import { ref, onMounted } from 'vue';
import axios from 'axios';

// URL of our backend API
const API_URL_SCORES = 'http://localhost:3000/api/scores';
const API_URL_QUESTIONS = 'http://localhost:3000/api/questions';

// Reactive state to hold the user's data and app status.
const questions = ref([]);
const userAnswers = ref({});
const userName = ref('');
const score = ref(null);
const message = ref('');
const highScores = ref([]);

// Fetch the questions
const fetchQuestions = async () => {
  try {
    const response = await axios.get(API_URL_QUESTIONS);
    questions.value = response.data;
  } catch (error) {
    console.error('Failed to fetch questions:', error);
    message.value = "Failed to load questions. Please check the server.";
  }
};

// Fetch latest high scores from backend
const fetchHighScores = async () => {
  try {
    const response = await axios.get(API_URL_SCORES);
    highScores.value = response.data;
  } catch (error) {
    console.error('Failed to fetch high scores:', error);
    message.value = "Failed to load high scores. Please check the server.";
  }
};

// Calculate the score and submit it to the backend
const calculateScore = async () => {
  // Check is user name is written
  if (!userName.value) {
    message.value = "Please enter your name before submitting.";
    return;
  }

  // Check if all questions are answered
  const totalQuestions = questions.value.length;
  const answeredQuestions = Object.keys(userAnswers.value).length;

  if (answeredQuestions < totalQuestions) {
    message.value = `Please complete all questions before submitting. You have answered ${answeredQuestions} out of ${totalQuestions} questions.`;
    return;
  }

  // Submit name and score to backend
  try {
    const response = await axios.post(API_URL_SCORES, {
      name: userName.value,
      userAnswers: userAnswers.value
    });

    score.value = response.data.score;
    message.value = response.data.message;
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
    message.value = "Failed to submit score. Please try again.";
  }
};

// Resets the worksheet to its initial state.
const resetWorksheet = () => {
  userName.value = '';
  score.value = null;
  message.value = '';
  userAnswers.value = {};
};

// lifecycle hook calling automatically
// `onMounted` is a Vue lifecycle hook we call it below other to list it as soon as we call it the app starts
// Fetch questions and scores when the component is mounted
onMounted(async () => {
  await fetchQuestions();
  await fetchHighScores();
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
              <button @click="calculateScore" class="glass-btn primary-action-btn">
                <span class="btn-text">Submit Answers</span>
              </button>
              <button @click="resetWorksheet" class="glass-btn secondary-action-btn">
                <span class="btn-text">Reset All</span>
              </button>
            </div>
          </div>
          <div class="controls-right">
            <div class="name-input-row">
              <label for="userName" class="input-label">Your Name</label>
              <input id="userName" v-model="userName" type="text" class="glass-input" placeholder="Enter your name..."
                maxlength="50" />
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

        <div class="questions-container">
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
            <h3 class="section-title">
              Highscores Top 10
            </h3>
          </div>

          <div v-if="highScores.length === 0" class="empty-leaderboard">
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

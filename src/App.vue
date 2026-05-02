<template>
  <div class="memory-game">
    <div class="game-header">
      <h1 class="game-title">记忆翻牌游戏</h1>
      
      <div class="game-controls">
        <div class="difficulty-selector">
          <label for="difficulty">难度选择：</label>
          <select id="difficulty" v-model="selectedDifficulty" @change="restartGame">
            <option value="4">4×4 (简单)</option>
            <option value="6">6×6 (中等)</option>
          </select>
        </div>
        
        <button class="restart-btn" @click="restartGame">
          重新开始
        </button>
      </div>
      
      <div class="game-stats">
        <div class="stat-item">
          <span class="stat-label">时间：</span>
          <span class="stat-value">{{ formatTime(elapsedTime) }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">翻牌次数：</span>
          <span class="stat-value">{{ flipCount }}</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">配对进度：</span>
          <span class="stat-value">{{ matchedPairs }} / {{ totalPairs }}</span>
        </div>
      </div>
    </div>
    
    <div 
      class="game-board" 
      :class="`board-${boardSize}`"
    >
      <div
        v-for="card in cards"
        :key="card.id"
        class="card"
        :class="{
          'flipped': card.isFlipped,
          'matched': card.isMatched,
          'disabled': isProcessing || card.isFlipped || card.isMatched
        }"
        @click="handleCardClick(card)"
      >
        <div class="card-inner">
          <div class="card-back"></div>
          <div class="card-front">
            <span class="card-emoji">{{ card.value }}</span>
          </div>
        </div>
      </div>
    </div>
    
    <div v-if="gameWon" class="game-overlay">
      <div class="game-message">
        <h2>🎉 恭喜你赢了！</h2>
        <div class="game-results">
          <p>完成时间：{{ formatTime(elapsedTime) }}</p>
          <p>翻牌次数：{{ flipCount }}</p>
        </div>
        <button class="play-again-btn" @click="restartGame">
          再玩一次
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>import { ref, computed, onMounted, onUnmounted } from 'vue';

const cardEmojis = [
  '🎈', '🎨', '🎭', '🎪', '🎯', '🎲',
  '🎸', '🎹', '🎺', '🎻', '🎼', '🎵',
  '🎾', '🎿', '🏀', '🏈', '⚽', '⚾',
  '🍎', '🍊', '🍋', '🍌', '🍉', '🍇',
  '🐶', '🐱', '🐭', '🐹', '🐰', '🦊',
  '🌟', '🌙', '⭐', '☀️', '🌈', '☁️'
];

const selectedDifficulty = ref('4');
const cards = ref([]);
const flippedCards = ref([]);
const matchedPairs = ref(0);
const flipCount = ref(0);
const elapsedTime = ref(0);
const gameWon = ref(false);
const isProcessing = ref(false);

let timerInterval = null;

const boardSize = computed(() => {
  return parseInt(selectedDifficulty.value);
});

const totalPairs = computed(() => {
  return (boardSize.value * boardSize.value) / 2;
});

const createCards = () => {
  const pairsNeeded = totalPairs.value;
  const selectedEmojis = [...cardEmojis].slice(0, pairsNeeded);
  const cardPairs = [...selectedEmojis, ...selectedEmojis];
  
  return cardPairs
    .sort(() => Math.random() - 0.5)
    .map((value, index) => ({
      id: index,
      value,
      isFlipped: false,
      isMatched: false
    }));
};

const startTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval);
  }
  elapsedTime.value = 0;
  timerInterval = setInterval(() => {
    elapsedTime.value++;
  }, 1000);
};

const stopTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval);
    timerInterval = null;
  }
};

const formatTime = (seconds) => {
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
};

const handleCardClick = (card) => {
  if (isProcessing.value || card.isFlipped || card.isMatched) {
    return;
  }
  
  card.isFlipped = true;
  flippedCards.value.push(card);
  
  if (flippedCards.value.length === 1 && flipCount.value === 0) {
    startTimer();
  }
  
  flipCount.value++;
  
  if (flippedCards.value.length === 2) {
    isProcessing.value = true;
    checkMatch();
  }
};

const checkMatch = () => {
  const [card1, card2] = flippedCards.value;
  
  if (card1.value === card2.value) {
    card1.isMatched = true;
    card2.isMatched = true;
    matchedPairs.value++;
    
    flippedCards.value = [];
    isProcessing.value = false;
    
    if (matchedPairs.value === totalPairs.value) {
      stopTimer();
      gameWon.value = true;
    }
  } else {
    setTimeout(() => {
      card1.isFlipped = false;
      card2.isFlipped = false;
      flippedCards.value = [];
      isProcessing.value = false;
    }, 1000);
  }
};

const restartGame = () => {
  stopTimer();
  cards.value = createCards();
  flippedCards.value = [];
  matchedPairs.value = 0;
  flipCount.value = 0;
  elapsedTime.value = 0;
  gameWon.value = false;
  isProcessing.value = false;
};

onMounted(() => {
  restartGame();
});

onUnmounted(() => {
  stopTimer();
});
</script>

<style scoped>
.memory-game {
  text-align: center;
}

.game-header {
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  padding: 24px;
  margin-bottom: 24px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
}

.game-title {
  color: #667eea;
  font-size: 2.5rem;
  margin-bottom: 20px;
  font-weight: 700;
}

.game-controls {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 20px;
  margin-bottom: 20px;
  flex-wrap: wrap;
}

.difficulty-selector {
  display: flex;
  align-items: center;
  gap: 8px;
}

.difficulty-selector label {
  font-weight: 600;
  color: #333;
}

.difficulty-selector select {
  padding: 8px 16px;
  border: 2px solid #667eea;
  border-radius: 8px;
  background: white;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.difficulty-selector select:focus {
  outline: none;
  border-color: #764ba2;
  box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.2);
}

.restart-btn {
  padding: 10px 24px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
}

.restart-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.restart-btn:active {
  transform: translateY(0);
}

.game-stats {
  display: flex;
  justify-content: center;
  gap: 30px;
  flex-wrap: wrap;
}

.stat-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.stat-label {
  font-size: 0.875rem;
  color: #666;
  font-weight: 500;
}

.stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #667eea;
}

.game-board {
  display: grid;
  gap: 12px;
  justify-content: center;
  padding: 20px;
}

.board-4 {
  grid-template-columns: repeat(4, 1fr);
  max-width: 500px;
  margin: 0 auto;
}

.board-6 {
  grid-template-columns: repeat(6, 1fr);
  max-width: 600px;
  margin: 0 auto;
}

.card {
  aspect-ratio: 1;
  perspective: 1000px;
  cursor: pointer;
}

.card.disabled {
  cursor: not-allowed;
}

.card-inner {
  position: relative;
  width: 100%;
  height: 100%;
  transition: transform 0.6s;
  transform-style: preserve-3d;
}

.card.flipped .card-inner,
.card.matched .card-inner {
  transform: rotateY(180deg);
}

.card-back,
.card-front {
  position: absolute;
  width: 100%;
  height: 100%;
  backface-visibility: hidden;
  border-radius: 12px;
  display: flex;
  justify-content: center;
  align-items: center;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.2);
}

.card-back {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border: 3px solid white;
}

.card-back::before {
  content: '?';
  font-size: 2.5rem;
  color: white;
  font-weight: 700;
}

.card-front {
  background: white;
  transform: rotateY(180deg);
  border: 3px solid #667eea;
}

.card-emoji {
  font-size: 2.5rem;
}

.card.matched .card-front {
  background: linear-gradient(135deg, #84fab0 0%, #8fd3f4 100%);
  border-color: #4facfe;
  animation: pulse 0.5s ease;
}

@keyframes pulse {
  0% {
    transform: rotateY(180deg) scale(1);
  }
  50% {
    transform: rotateY(180deg) scale(1.1);
  }
  100% {
    transform: rotateY(180deg) scale(1);
  }
}

.game-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.game-message {
  background: white;
  padding: 40px;
  border-radius: 20px;
  text-align: center;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.3);
  animation: slideIn 0.5s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-50px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.game-message h2 {
  color: #667eea;
  font-size: 2rem;
  margin-bottom: 20px;
}

.game-results {
  margin-bottom: 24px;
}

.game-results p {
  font-size: 1.2rem;
  color: #333;
  margin: 12px 0;
  font-weight: 500;
}

.play-again-btn {
  padding: 12px 32px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 1.1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.play-again-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

@media (max-width: 600px) {
  .game-title {
    font-size: 1.8rem;
  }
  
  .game-stats {
    gap: 15px;
  }
  
  .stat-value {
    font-size: 1.2rem;
  }
  
  .board-4 {
    max-width: 350px;
  }
  
  .board-6 {
    max-width: 400px;
  }
  
  .card-emoji {
    font-size: 1.8rem;
  }
  
  .card-back::before {
    font-size: 1.8rem;
  }
}
</style>

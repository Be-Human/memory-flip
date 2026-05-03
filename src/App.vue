<template>
  <div class="memory-game">
    <div class="game-header">
      <div class="header-top">
        <h1 class="game-title">记忆翻牌游戏</h1>
        <button class="sound-btn" @click="toggleSound" :title="isMuted ? '取消静音' : '静音'">
          {{ isMuted ? '🔇' : '🔊' }}
        </button>
      </div>
      
      <div class="game-controls">
        <div class="difficulty-selector">
          <label for="difficulty">难度选择：</label>
          <select id="difficulty" v-model="selectedDifficulty" @change="restartGame">
            <option value="4">4×4 (简单)</option>
            <option value="6">6×6 (中等)</option>
          </select>
        </div>
        
        <div class="theme-selector">
          <label for="theme">主题选择：</label>
          <select id="theme" v-model="selectedTheme" @change="handleThemeChange">
            <option value="custom">🖼️ 自定义图片</option>
            <option v-for="(theme, key) in themes" :key="key" :value="key">
              {{ theme.name }}
            </option>
          </select>
        </div>
        
        <button class="restart-btn" @click="restartGame">
          重新开始
        </button>
      </div>
      
      <div v-if="selectedTheme === 'custom'" class="custom-upload-section">
        <div class="upload-area" @click="triggerFileInput" @dragover.prevent @drop.prevent="handleDrop">
          <input 
            type="file" 
            ref="fileInputRef" 
            accept="image/*" 
            multiple 
            @change="handleFileSelect"
            style="display: none;"
          />
          <div class="upload-icon">📁</div>
          <p class="upload-text">点击或拖拽图片到这里上传</p>
          <p class="upload-hint">当前难度需要至少 {{ totalPairs }} 张不同图片</p>
        </div>
        
        <div v-if="customImages.length > 0" class="uploaded-images">
          <p class="images-count">已上传 {{ customImages.length }} 张图片</p>
          <div class="images-grid">
            <div v-for="(img, index) in customImages" :key="index" class="image-preview">
              <img :src="img.url" alt="预览" class="preview-img" />
              <button class="remove-img-btn" @click="removeImage(index)" title="删除">×</button>
            </div>
          </div>
          <button v-if="customImages.length > 0" class="clear-all-btn" @click="clearAllImages">
            清除全部
          </button>
        </div>
        
        <div v-if="selectedTheme === 'custom' && customImages.length < totalPairs" class="warning-message">
          ⚠️ 图片不足！当前难度 {{ selectedDifficulty }}×{{ selectedDifficulty }} 需要至少 {{ totalPairs }} 张图片
        </div>
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
      
      <div class="best-records" v-if="currentBestRecord">
        <h3 class="records-title">🏆 当前难度最佳记录</h3>
        <div class="records-content">
          <div class="record-item">
            <span class="record-label">最快时间：</span>
            <span class="record-value">{{ currentBestRecord.bestTime ? formatTime(currentBestRecord.bestTime) : '暂无记录' }}</span>
          </div>
          <div class="record-item">
            <span class="record-label">最少翻牌：</span>
            <span class="record-value">{{ currentBestRecord.bestFlips || '暂无记录' }}</span>
          </div>
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
            <template v-if="card.isImage">
              <img :src="card.value" alt="卡片" class="card-image" />
            </template>
            <template v-else>
              <span class="card-emoji">{{ card.value }}</span>
            </template>
          </div>
        </div>
      </div>
    </div>
    
    <div v-if="gameWon" class="game-overlay">
      <div class="game-message">
        <h2>🎉 恭喜你赢了！</h2>
        
        <div v-if="newRecordType" class="new-record-banner">
          <span class="new-record-text">
            🏆 新纪录！
            <span v-if="newRecordType === 'time'">时间：{{ formatTime(elapsedTime) }}</span>
            <span v-else-if="newRecordType === 'flips'">翻牌次数：{{ flipCount }}</span>
            <span v-else>时间：{{ formatTime(elapsedTime) }} | 翻牌次数：{{ flipCount }}</span>
          </span>
        </div>
        
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

<script setup>import { ref, computed, onMounted, onUnmounted, watch } from 'vue';

const themes = {
  animals: {
    name: '🐾 动物',
    emojis: ['🐶', '🐱', '🐭', '🐹', '🐰', '🦊', '🐻', '🐼', '🐨', '🐯', '🦁', '🐮', '🐷', '🐸', '🐵', '🐔', '🐧', '🦆']
  },
  food: {
    name: '🍎 食物',
    emojis: ['🍎', '🍊', '🍋', '🍌', '🍉', '🍇', '🍓', '🍒', '🍑', '🥝', '🍍', '🍅', '🥕', '🌽', '🍕', '🍔', '🍟', '🍦']
  },
  sports: {
    name: '⚽ 运动',
    emojis: ['⚽', '🏀', '🏈', '⚾', '🎾', '🏐', '🏉', '🎱', '🏓', '🏸', '🏒', '🥊', '⛳', '🎣', '🏹', '🎿', '⛷️', '🏂']
  },
  music: {
    name: '🎵 音乐',
    emojis: ['🎵', '🎶', '🎸', '🎹', '🎺', '🎻', '🥁', '🎷', '🎬', '🎭', '🎪', '🎨', '🎰', '🎲', '🎯', '🎳', '🎮', '🎰']
  },
  nature: {
    name: '🌸 自然',
    emojis: ['🌸', '🌺', '🌻', '🌹', '🌷', '🌼', '🌲', '🌳', '🌴', '🌵', '🍀', '🌿', '🌾', '🍁', '🍂', '🌙', '⭐', '☀️']
  }
};

const STORAGE_KEY = 'memory-flip-records';
const IMAGES_STORAGE_KEY = 'memory-flip-custom-images';

const selectedDifficulty = ref('4');
const selectedTheme = ref('animals');
const cards = ref([]);
const flippedCards = ref([]);
const matchedPairs = ref(0);
const flipCount = ref(0);
const elapsedTime = ref(0);
const gameWon = ref(false);
const isProcessing = ref(false);
const newRecordType = ref(null);
const bestRecords = ref({});
const customImages = ref([]);
const fileInputRef = ref(null);
const isMuted = ref(false);
const audioContext = ref(null);

let timerInterval = null;

const boardSize = computed(() => {
  return parseInt(selectedDifficulty.value);
});

const totalPairs = computed(() => {
  return (boardSize.value * boardSize.value) / 2;
});

const currentBestRecord = computed(() => {
  return bestRecords.value[selectedDifficulty.value] || { bestTime: null, bestFlips: null };
});

const loadRecords = () => {
  try {
    const saved = localStorage.getItem(STORAGE_KEY);
    if (saved) {
      bestRecords.value = JSON.parse(saved);
    }
  } catch (error) {
    console.error('Failed to load records:', error);
    bestRecords.value = {};
  }
};

const saveRecords = () => {
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(bestRecords.value));
  } catch (error) {
    console.error('Failed to save records:', error);
  }
};

const checkAndUpdateRecord = () => {
  const difficulty = selectedDifficulty.value;
  const currentTime = elapsedTime.value;
  const currentFlips = flipCount.value;
  
  let isNewRecord = false;
  let recordType = null;
  
  if (!bestRecords.value[difficulty]) {
    bestRecords.value[difficulty] = {
      bestTime: currentTime,
      bestFlips: currentFlips
    };
    isNewRecord = true;
    recordType = 'both';
  } else {
    const record = bestRecords.value[difficulty];
    const timeBetter = record.bestTime === null || currentTime < record.bestTime;
    const flipsBetter = record.bestFlips === null || currentFlips < record.bestFlips;
    
    if (timeBetter) {
      record.bestTime = currentTime;
      isNewRecord = true;
      recordType = 'time';
    }
    
    if (flipsBetter) {
      record.bestFlips = currentFlips;
      isNewRecord = true;
      recordType = recordType ? 'both' : 'flips';
    }
  }
  
  if (isNewRecord) {
    newRecordType.value = recordType;
    saveRecords();
  }
  
  return isNewRecord;
};

const createCards = () => {
  const pairsNeeded = totalPairs.value;
  let selectedValues = [];
  let isImage = false;
  
  if (selectedTheme.value === 'custom') {
    if (customImages.value.length >= pairsNeeded) {
      selectedValues = customImages.value.slice(0, pairsNeeded).map(img => img.url);
      isImage = true;
    } else {
      return [];
    }
  } else {
    const themeEmojis = themes[selectedTheme.value].emojis;
    selectedValues = [...themeEmojis].slice(0, pairsNeeded);
  }
  
  const cardPairs = [...selectedValues, ...selectedValues];
  
  return cardPairs
    .sort(() => Math.random() - 0.5)
    .map((value, index) => ({
      id: index,
      value,
      isImage,
      isFlipped: false,
      isMatched: false
    }));
};

const initAudio = () => {
  if (!audioContext.value) {
    try {
      audioContext.value = new (window.AudioContext || window.webkitAudioContext)();
    } catch (e) {
      console.log('Web Audio API not supported');
    }
  }
  if (audioContext.value && audioContext.value.state === 'suspended') {
    audioContext.value.resume();
  }
};

const playFlipSound = () => {
  if (isMuted.value || !audioContext.value) return;
  
  const oscillator = audioContext.value.createOscillator();
  const gainNode = audioContext.value.createGain();
  
  oscillator.connect(gainNode);
  gainNode.connect(audioContext.value.destination);
  
  oscillator.type = 'sine';
  oscillator.frequency.setValueAtTime(800, audioContext.value.currentTime);
  oscillator.frequency.exponentialRampToValueAtTime(400, audioContext.value.currentTime + 0.1);
  
  gainNode.gain.setValueAtTime(0.15, audioContext.value.currentTime);
  gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.value.currentTime + 0.1);
  
  oscillator.start(audioContext.value.currentTime);
  oscillator.stop(audioContext.value.currentTime + 0.1);
};

const playMatchSound = () => {
  if (isMuted.value || !audioContext.value) return;
  
  const notes = [523.25, 659.25, 783.99];
  
  notes.forEach((freq, index) => {
    const oscillator = audioContext.value.createOscillator();
    const gainNode = audioContext.value.createGain();
    
    oscillator.connect(gainNode);
    gainNode.connect(audioContext.value.destination);
    
    oscillator.type = 'sine';
    oscillator.frequency.setValueAtTime(freq, audioContext.value.currentTime + index * 0.08);
    
    gainNode.gain.setValueAtTime(0.15, audioContext.value.currentTime + index * 0.08);
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.value.currentTime + index * 0.08 + 0.2);
    
    oscillator.start(audioContext.value.currentTime + index * 0.08);
    oscillator.stop(audioContext.value.currentTime + index * 0.08 + 0.2);
  });
};

const playWinSound = () => {
  if (isMuted.value || !audioContext.value) return;
  
  const notes = [523.25, 659.25, 783.99, 1046.50, 783.99, 1046.50];
  
  notes.forEach((freq, index) => {
    const oscillator = audioContext.value.createOscillator();
    const gainNode = audioContext.value.createGain();
    
    oscillator.connect(gainNode);
    gainNode.connect(audioContext.value.destination);
    
    oscillator.type = 'sine';
    oscillator.frequency.setValueAtTime(freq, audioContext.value.currentTime + index * 0.12);
    
    gainNode.gain.setValueAtTime(0.15, audioContext.value.currentTime + index * 0.12);
    gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.value.currentTime + index * 0.12 + 0.25);
    
    oscillator.start(audioContext.value.currentTime + index * 0.12);
    oscillator.stop(audioContext.value.currentTime + index * 0.12 + 0.25);
  });
};

const toggleSound = () => {
  isMuted.value = !isMuted.value;
  initAudio();
  if (!isMuted.value) {
    playFlipSound();
  }
};

const handleThemeChange = () => {
  if (selectedTheme.value === 'custom') {
    if (customImages.value.length < totalPairs.value) {
      restartGame();
    } else {
      restartGame();
    }
  } else {
    restartGame();
  }
};

const triggerFileInput = () => {
  initAudio();
  fileInputRef.value.click();
};

const handleFileSelect = (event) => {
  const files = Array.from(event.target.files);
  processFiles(files);
  event.target.value = '';
};

const handleDrop = (event) => {
  const files = Array.from(event.dataTransfer.files).filter(file => file.type.startsWith('image/'));
  processFiles(files);
};

const processFiles = (files) => {
  const existingUrls = new Set(customImages.value.map(img => img.url));
  const existingNames = new Set(customImages.value.map(img => img.name));
  const newlyAddedUrls = new Set();
  const newlyAddedNames = new Set();
  
  files.forEach(file => {
    if (existingNames.has(file.name) || newlyAddedNames.has(file.name)) {
      console.log(`跳过已存在的图片: ${file.name}`);
      return;
    }
    
    const reader = new FileReader();
    reader.onload = (e) => {
      const imageUrl = e.target.result;
      
      if (existingUrls.has(imageUrl) || newlyAddedUrls.has(imageUrl)) {
        console.log(`跳过内容相同的图片: ${file.name}`);
        return;
      }
      
      customImages.value.push({
        id: Date.now() + Math.random(),
        url: imageUrl,
        name: file.name
      });
      
      newlyAddedUrls.add(imageUrl);
      newlyAddedNames.add(file.name);
      saveCustomImages();
      
      if (selectedTheme.value === 'custom' && customImages.value.length >= totalPairs.value) {
        restartGame();
      }
    };
    reader.readAsDataURL(file);
  });
};

const removeImage = (index) => {
  customImages.value.splice(index, 1);
  saveCustomImages();
  if (selectedTheme.value === 'custom') {
    restartGame();
  }
};

const clearAllImages = () => {
  customImages.value = [];
  saveCustomImages();
  if (selectedTheme.value === 'custom') {
    restartGame();
  }
};

const saveCustomImages = () => {
  try {
    localStorage.setItem(IMAGES_STORAGE_KEY, JSON.stringify(customImages.value));
  } catch (error) {
    console.error('Failed to save images:', error);
  }
};

const loadCustomImages = () => {
  try {
    const saved = localStorage.getItem(IMAGES_STORAGE_KEY);
    if (saved) {
      customImages.value = JSON.parse(saved);
    }
  } catch (error) {
    console.error('Failed to load images:', error);
    customImages.value = [];
  }
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
  
  initAudio();
  playFlipSound();
  
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
    
    playMatchSound();
    
    flippedCards.value = [];
    isProcessing.value = false;
    
    if (matchedPairs.value === totalPairs.value) {
      stopTimer();
      checkAndUpdateRecord();
      setTimeout(() => {
        playWinSound();
      }, 300);
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
  newRecordType.value = null;
  cards.value = createCards();
  flippedCards.value = [];
  matchedPairs.value = 0;
  flipCount.value = 0;
  elapsedTime.value = 0;
  gameWon.value = false;
  isProcessing.value = false;
};

onMounted(() => {
  loadRecords();
  loadCustomImages();
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

.header-top {
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  margin-bottom: 16px;
}

.sound-btn {
  position: absolute;
  right: 0;
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  padding: 8px;
  border-radius: 8px;
  transition: all 0.3s ease;
}

.sound-btn:hover {
  background: rgba(102, 126, 234, 0.1);
  transform: scale(1.1);
}

.game-title {
  color: #667eea;
  font-size: 2.5rem;
  margin-bottom: 0;
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

.theme-selector {
  display: flex;
  align-items: center;
  gap: 8px;
}

.theme-selector label {
  font-weight: 600;
  color: #333;
}

.theme-selector select {
  padding: 8px 16px;
  border: 2px solid #667eea;
  border-radius: 8px;
  background: white;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s ease;
}

.theme-selector select:focus {
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

.custom-upload-section {
  margin-top: 16px;
  padding: 16px;
  background: rgba(102, 126, 234, 0.05);
  border-radius: 12px;
  border: 2px dashed rgba(102, 126, 234, 0.3);
}

.upload-area {
  border: 3px dashed #667eea;
  border-radius: 12px;
  padding: 24px;
  cursor: pointer;
  transition: all 0.3s ease;
  background: rgba(255, 255, 255, 0.8);
}

.upload-area:hover {
  border-color: #764ba2;
  background: rgba(102, 126, 234, 0.1);
}

.upload-icon {
  font-size: 3rem;
  margin-bottom: 8px;
}

.upload-text {
  font-size: 1.1rem;
  font-weight: 600;
  color: #667eea;
  margin: 0 0 4px 0;
}

.upload-hint {
  font-size: 0.875rem;
  color: #666;
  margin: 0;
}

.uploaded-images {
  margin-top: 16px;
}

.images-count {
  font-size: 0.95rem;
  color: #333;
  margin-bottom: 12px;
  font-weight: 600;
}

.images-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  justify-content: center;
  margin-bottom: 12px;
}

.image-preview {
  position: relative;
  width: 60px;
  height: 60px;
  border-radius: 8px;
  overflow: hidden;
  border: 2px solid #667eea;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
}

.preview-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.remove-img-btn {
  position: absolute;
  top: -8px;
  right: -8px;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #f5576c;
  color: white;
  border: none;
  font-size: 1rem;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0;
  line-height: 1;
  transition: all 0.2s ease;
}

.remove-img-btn:hover {
  background: #e0465a;
  transform: scale(1.1);
}

.clear-all-btn {
  padding: 8px 20px;
  background: #f5576c;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.clear-all-btn:hover {
  background: #e0465a;
  transform: translateY(-1px);
}

.warning-message {
  margin-top: 12px;
  padding: 12px;
  background: rgba(245, 87, 108, 0.1);
  border: 2px solid #f5576c;
  border-radius: 8px;
  color: #e0465a;
  font-weight: 600;
  font-size: 0.95rem;
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

.best-records {
  margin-top: 20px;
  padding: 16px 20px;
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.1) 0%, rgba(118, 75, 162, 0.1) 100%);
  border-radius: 12px;
  border: 2px dashed rgba(102, 126, 234, 0.4);
}

.records-title {
  margin: 0 0 12px 0;
  font-size: 1.1rem;
  color: #667eea;
  font-weight: 600;
}

.records-content {
  display: flex;
  justify-content: center;
  gap: 40px;
  flex-wrap: wrap;
}

.record-item {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.record-label {
  font-size: 0.875rem;
  color: #666;
  font-weight: 500;
}

.record-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #764ba2;
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

.card-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 8px;
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

.new-record-banner {
  background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
  color: white;
  padding: 16px 24px;
  border-radius: 12px;
  margin-bottom: 20px;
  animation: pulseRecord 1s ease infinite;
  box-shadow: 0 4px 15px rgba(245, 87, 108, 0.4);
}

.new-record-text {
  font-size: 1.1rem;
  font-weight: 600;
}

@keyframes pulseRecord {
  0%, 100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.02);
  }
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
  .header-top {
    margin-bottom: 12px;
  }
  
  .sound-btn {
    position: static;
    margin-left: auto;
    font-size: 1.3rem;
  }
  
  .game-title {
    font-size: 1.8rem;
    flex: 1;
    text-align: left;
  }
  
  .game-controls {
    flex-direction: column;
    gap: 12px;
  }
  
  .difficulty-selector,
  .theme-selector {
    flex-direction: column;
    gap: 6px;
  }
  
  .custom-upload-section {
    padding: 12px;
    margin-top: 12px;
  }
  
  .upload-area {
    padding: 16px;
  }
  
  .upload-icon {
    font-size: 2.5rem;
  }
  
  .upload-text {
    font-size: 1rem;
  }
  
  .image-preview {
    width: 50px;
    height: 50px;
  }
  
  .game-stats {
    gap: 15px;
  }
  
  .stat-value {
    font-size: 1.2rem;
  }
  
  .best-records {
    padding: 12px 16px;
  }
  
  .records-title {
    font-size: 1rem;
  }
  
  .records-content {
    gap: 20px;
  }
  
  .record-value {
    font-size: 1.1rem;
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
  
  .game-message {
    padding: 30px 20px;
    margin: 20px;
  }
  
  .game-message h2 {
    font-size: 1.5rem;
  }
  
  .new-record-text {
    font-size: 1rem;
  }
}
</style>

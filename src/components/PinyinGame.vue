<script setup>
import { ref, computed } from 'vue';
import { Volume2, ChevronLeft, ChevronRight, Table } from 'lucide-vue-next';

const emit = defineEmits(['back']);

// Pinyin data structure
const initials = [
  'b', 'p', 'm', 'f', 'd', 't', 'n', 'l', 'g', 'k', 'h',
  'j', 'q', 'x', 'zh', 'ch', 'sh', 'r', 'z', 'c', 's',
  'y', 'w'
];

const finals = [
  'a', 'o', 'e', 'i', 'u', 'ü', 'ai', 'ei', 'ui', 'ao', 'ou',
  'iu', 'ie', 'üe', 'er', 'an', 'en', 'in', 'un', 'ün', 'ang',
  'eng', 'ing', 'ong'
];

// Tone marks mapping
const toneMarks = {
  'a': ['ā', 'á', 'ǎ', 'à'],
  'o': ['ō', 'ó', 'ǒ', 'ò'],
  'e': ['ē', 'é', 'ě', 'è'],
  'i': ['ī', 'í', 'ǐ', 'ì'],
  'u': ['ū', 'ú', 'ǔ', 'ù'],
  'ü': ['ǖ', 'ǘ', 'ǚ', 'ǜ']
};

// Combine all pinyin sounds into a flat list for navigation
const allSounds = computed(() => {
  const list = [];
  
  list.push({ type: 'header', title: '声母 (Initials)' });
  initials.forEach(s => list.push({ type: 'initial', char: s, label: s }));
  
  list.push({ type: 'header', title: '韵母 (Finals)' });
  finals.forEach(s => list.push({ type: 'final', char: s, label: s }));
  
  return list;
});

// Filter out headers for navigation index
const playableSounds = computed(() => allSounds.value.filter(item => item.type !== 'header'));

// Find index of 'a' to start with
const startIndex = computed(() => playableSounds.value.findIndex(item => item.char === 'a'));
const currentIndex = ref(startIndex.value !== -1 ? startIndex.value : 0);

const currentSound = computed(() => playableSounds.value[currentIndex.value]);

const currentTones = computed(() => {
  const char = currentSound.value.char;
  // Only show tones for single vowels that are in our map
  if (currentSound.value.type === 'final' && toneMarks[char]) {
    return toneMarks[char];
  }
  return [];
});

const playSound = (text) => {
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'zh-CN';
  utterance.rate = 0.8;
  window.speechSynthesis.speak(utterance);
};

const nextSound = () => {
  if (currentIndex.value < playableSounds.value.length - 1) {
    currentIndex.value++;
    playSound(currentSound.value.char);
  }
};

const prevSound = () => {
  if (currentIndex.value > 0) {
    currentIndex.value--;
    playSound(currentSound.value.char);
  }
};

const handleKeyDown = (e) => {
  if (e.key === 'ArrowRight') nextSound();
  if (e.key === 'ArrowLeft') prevSound();
  if (e.key === ' ' || e.key === 'Enter') playSound(currentSound.value.char);
};

// Auto focus container for key events
const containerRef = ref(null);
import { onMounted } from 'vue';
onMounted(() => {
  if (containerRef.value) containerRef.value.focus();
});
</script>

<template>
  <div 
    class="min-h-screen bg-indigo-50 flex flex-col items-center justify-center p-4 font-sans outline-none"
    tabindex="0"
    ref="containerRef"
    @keydown="handleKeyDown"
  >
    <div class="bg-white p-8 rounded-2xl shadow-xl max-w-md w-full text-center relative">
      <div class="absolute top-4 left-4">
        <button 
          @click="$emit('back')"
          class="p-2 text-gray-500 hover:text-gray-800 transition-colors flex items-center gap-2"
        >
          ← 返回
        </button>
      </div>
      
      <div class="mb-6 flex justify-center">
        <div class="w-20 h-20 bg-indigo-100 rounded-full flex items-center justify-center text-indigo-600">
          <Table :size="40" />
        </div>
      </div>
      <h1 class="text-3xl font-bold text-gray-800 mb-2">拼音表</h1>
      <p class="text-gray-500 mb-8">声母与韵母学习</p>

      <div class="bg-indigo-50 rounded-2xl p-8 mb-8 flex flex-col items-center justify-center min-h-[300px]">
        <div class="text-sm text-indigo-500 font-bold mb-4 uppercase tracking-wider">
          {{ currentSound.type === 'initial' ? '声母 (Initial)' : '韵母 (Final)' }}
        </div>
        
        <div class="text-8xl font-bold text-gray-800 mb-8 font-serif">
          {{ currentSound.char }}
        </div>

        <!-- Tone variations -->
        <div v-if="currentTones.length > 0" class="flex gap-4">
          <div 
            v-for="(tone, idx) in currentTones" 
            :key="idx"
            @click.stop="playSound(tone)"
            class="w-16 h-16 rounded-xl bg-white border-2 border-indigo-100 flex items-center justify-center text-2xl font-bold text-indigo-600 cursor-pointer hover:border-indigo-500 hover:shadow-md transition-all active:scale-95"
          >
            {{ tone }}
          </div>
        </div>
      </div>

      <!-- Controls -->
      <div class="flex justify-between items-center gap-4">
        <button 
          @click="prevSound"
          :disabled="currentIndex === 0"
          class="p-4 rounded-full bg-gray-100 text-gray-600 hover:bg-gray-200 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
        >
          <ChevronLeft :size="32" />
        </button>

        <button 
          @click="playSound(currentSound.char)"
          class="w-20 h-20 rounded-full bg-indigo-600 text-white flex items-center justify-center hover:bg-indigo-700 transition-transform active:scale-95 shadow-lg shadow-indigo-200"
        >
          <Volume2 :size="40" />
        </button>

        <button 
          @click="nextSound"
          :disabled="currentIndex === playableSounds.length - 1"
          class="p-4 rounded-full bg-gray-100 text-gray-600 hover:bg-gray-200 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
        >
          <ChevronRight :size="32" />
        </button>
      </div>
      
      <div class="mt-6 text-gray-400 text-sm">
        提示: 可使用键盘左右方向键切换
      </div>
    </div>
  </div>
</template>

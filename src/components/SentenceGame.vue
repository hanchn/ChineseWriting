<script setup>
import { ref, onMounted, computed } from 'vue';
import { Volume2, RefreshCw, ArrowRight, Eye, Book } from 'lucide-vue-next';

const emit = defineEmits(['back']);

const words = ref([]);
const currentIndex = ref(0);
const loading = ref(false);
const gameState = ref('start'); // start, playing, finished
const isRevealed = ref(false);

const structure = ref({});
const selectedBook = ref('');
const selectedFile = ref('');

onMounted(() => {
  fetchStructure();
});

const lessonOptions = computed(() => {
  const options = [];
  if (structure.value) {
    for (const [book, files] of Object.entries(structure.value)) {
      if (!book.toLowerCase().includes('sentences')) continue; // Filter for Sentences books

      if (Array.isArray(files)) {
        for (const file of files) {
          options.push({
            label: `${book} / ${file.replace('.json', '')}`,
            value: `${book}|${file}`
          });
        }
      }
    }
  }
  return options;
});

const currentSelection = ref('');

const handleSelectionChange = () => {
  if (!currentSelection.value) {
    selectedBook.value = '';
    selectedFile.value = '';
    return;
  }
  const [book, file] = currentSelection.value.split('|');
  selectedBook.value = book;
  selectedFile.value = file;
};

const fetchStructure = async () => {
  try {
    const res = await fetch('http://localhost:3003/api/structure');
    const data = await res.json();
    structure.value = data;
    
    // Auto-select first lesson
    if (lessonOptions.value.length > 0) {
      currentSelection.value = lessonOptions.value[0].value;
      handleSelectionChange();
    }
  } catch (error) {
    console.error('Failed to fetch structure', error);
  }
};

const startGame = async () => {
  loading.value = true;
  try {
    let url = 'http://localhost:3003/api/words';
    const params = new URLSearchParams();
    
    if (selectedBook.value) {
      params.append('book', selectedBook.value);
      if (selectedFile.value) {
        params.append('file', selectedFile.value);
      }
    }
    
    const res = await fetch(`${url}?${params.toString()}`);
    const data = await res.json();
    
    if (data.length === 0) {
      alert('No sentences found!');
      loading.value = false;
      return;
    }

    let shuffled = data.sort(() => Math.random() - 0.5);
    words.value = shuffled;
    currentIndex.value = 0;
    isRevealed.value = false;
    gameState.value = 'playing';
  } catch (error) {
    console.error('Failed to fetch words', error);
    alert('Failed to start game');
  } finally {
    loading.value = false;
  }
};

const speakWord = (text) => {
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'zh-CN';
  // Slower rate for sentences
  utterance.rate = 0.8;
  window.speechSynthesis.speak(utterance);
};

const handleReveal = () => {
  isRevealed.value = true;
  const currentWord = words.value[currentIndex.value];
  if (currentWord) {
    speakWord(currentWord.word);
  }
};

const nextWord = () => {
  if (currentIndex.value < words.value.length - 1) {
    currentIndex.value++;
    isRevealed.value = false;
  } else {
    gameState.value = 'finished';
  }
};

const currentWordObj = computed(() => words.value[currentIndex.value]);

const pinyin = computed(() => {
  if (!currentWordObj.value) return '';
  const def = currentWordObj.value.definition;
  // Try to split by " - " first
  if (def.includes(' - ')) {
    return def.split(' - ')[0];
  }
  return '';
});

const translation = computed(() => {
  if (!currentWordObj.value) return '';
  const def = currentWordObj.value.definition;
  if (def.includes(' - ')) {
    return def.split(' - ').slice(1).join(' - ');
  }
  return def;
});
</script>

<template>
  <div v-if="loading" class="min-h-screen flex items-center justify-center bg-amber-50 text-amber-800">Loading...</div>

  <!-- Start Screen -->
  <div v-else-if="gameState === 'start'" class="min-h-screen bg-gradient-to-br from-amber-50 to-orange-50 flex flex-col items-center justify-center p-4 font-sans">
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
        <div class="w-20 h-20 bg-amber-100 rounded-full flex items-center justify-center text-amber-600">
          <Book :size="40" />
        </div>
      </div>
      <h1 class="text-3xl font-bold text-gray-800 mb-2">句子阅读</h1>
      <p class="text-gray-500 mb-8">练习中文句子阅读</p>

      <div class="space-y-4 mb-8 text-left">
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">选择课程</label>
          <select 
            v-model="currentSelection"
            @change="handleSelectionChange"
            class="w-full p-3 bg-gray-50 border border-gray-200 rounded-lg focus:ring-2 focus:ring-amber-500 outline-none transition-all"
          >
            <option value="">请选择...</option>
            <option v-for="option in lessonOptions" :key="option.value" :value="option.value">
              {{ option.label }}
            </option>
          </select>
        </div>
      </div>

      <button 
        @click="startGame"
        class="w-full py-4 bg-amber-600 hover:bg-amber-700 text-white rounded-xl font-bold text-lg shadow-lg shadow-amber-200 transition-all active:scale-95"
      >
        开始阅读
      </button>
    </div>
  </div>

  <!-- Finished Screen -->
  <div v-else-if="gameState === 'finished'" class="min-h-screen bg-amber-50 flex flex-col items-center justify-center p-4 font-sans">
    <div class="max-w-md w-full bg-white rounded-2xl shadow-xl p-8 text-center">
       <h2 class="text-3xl font-bold text-gray-800 mb-4">完成!</h2>
       <p class="text-gray-600 mb-8">你已经阅读了 {{ words.length }} 个句子.</p>
       
       <button 
          @click="gameState = 'start'"
          class="px-6 py-3 bg-gray-800 text-white rounded-full font-medium hover:bg-gray-900 transition-colors flex items-center gap-2 mx-auto"
        >
          <RefreshCw :size="18" /> 返回菜单
        </button>
    </div>
  </div>

  <!-- Playing Screen -->
  <div v-else class="min-h-screen bg-amber-50 flex flex-col items-center justify-center p-4 font-sans">
    <div class="max-w-4xl w-full bg-white rounded-2xl shadow-xl p-8 relative min-h-[400px] flex flex-col">
      <!-- Progress -->
      <div class="absolute top-0 left-0 w-full h-2 bg-gray-100">
        <div 
          class="h-full bg-amber-500 transition-all duration-300"
          :style="{ width: `${((currentIndex + 1) / words.length) * 100}%` }"
        />
      </div>

      <div class="mt-4 flex justify-between items-center mb-4">
        <span class="text-gray-400 text-sm">第 {{ currentIndex + 1 }} 句 / 共 {{ words.length }} 句</span>
        <button 
          @click="gameState = 'start'" 
          class="text-sm text-gray-500 hover:text-gray-800 underline"
        >
          退出
        </button>
      </div>

      <!-- Card Content -->
      <div class="flex-1 flex flex-col items-center justify-center px-4">
        <!-- Sentence Display -->
        <div class="mb-8 text-center w-full">
          <h2 class="text-4xl md:text-5xl font-bold text-gray-800 mb-8 font-serif leading-relaxed">
            {{ currentWordObj?.word }}
          </h2>
        </div>

        <!-- Hidden/Revealed Info -->
        <div class="transition-opacity duration-500 text-center" :class="{ 'opacity-100': isRevealed, 'opacity-0': !isRevealed }">
          <p class="text-xl text-amber-700 font-medium mb-2">{{ pinyin }}</p>
          <p class="text-xl text-gray-600">{{ translation }}</p>
        </div>
      </div>

      <!-- Controls -->
      <div class="mt-8 flex justify-center items-center gap-6">
        <button v-if="!isRevealed"
          @click="handleReveal"
          class="px-8 py-4 bg-amber-600 hover:bg-amber-700 text-white rounded-full font-bold text-xl shadow-lg shadow-amber-200 transition-all active:scale-95 flex items-center gap-2"
        >
          <Eye :size="24" /> 显示答案 & 朗读
        </button>
        <template v-else>
          <button 
            @click="speakWord(currentWordObj.word)"
            class="w-16 h-16 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center hover:bg-blue-200 transition-transform active:scale-95 shadow-sm"
          >
            <Volume2 :size="32" />
          </button>
          
          <button 
            @click="nextWord"
            class="px-8 py-4 bg-green-500 hover:bg-green-600 text-white rounded-full font-bold text-xl shadow-lg shadow-green-200 transition-all active:scale-95 flex items-center gap-2"
          >
            下一句 <ArrowRight :size="24" />
          </button>
        </template>
      </div>
    </div>
  </div>
</template>

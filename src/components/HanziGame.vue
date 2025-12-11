<script setup>
import { ref, onMounted, computed, nextTick } from 'vue';
import { Volume2, RefreshCw, ArrowRight, Eye, BookOpen, PenTool } from 'lucide-vue-next';
import HanziWriter from 'hanzi-writer';

const emit = defineEmits(['back']);

const words = ref([]);
const currentIndex = ref(0);
const loading = ref(false);
const gameState = ref('start'); // start, playing, finished
const isRevealed = ref(false);
const writerRef = ref(null);
let writerInstances = [];
const isWriting = ref(false);

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
      if (!book.toLowerCase().includes('hanzi')) continue; // Filter for Hanzi books

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
      alert('No words found!');
      loading.value = false;
      return;
    }

    let shuffled = data.sort(() => Math.random() - 0.5);
    words.value = shuffled;
    currentIndex.value = 0;
    isRevealed.value = false;
    isWriting.value = false;
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
    isWriting.value = false;
    writerInstances = [];
  } else {
    gameState.value = 'finished';
  }
};

const showStrokeOrder = () => {
  isWriting.value = true;
  nextTick(async () => {
    if (writerRef.value) {
      // Clear previous content
      writerRef.value.innerHTML = '';
      writerInstances = [];
      
      const chars = currentWordObj.value.word.split('');
      
      for (const char of chars) {
        // Create a container for each character
        const charContainer = document.createElement('div');
        charContainer.style.width = '200px';
        charContainer.style.height = '200px';
        charContainer.style.backgroundColor = 'white';
        charContainer.style.borderRadius = '12px';
        charContainer.style.border = '1px solid #e5e7eb';
        // Add margin if there are multiple characters
        if (chars.length > 1) {
          charContainer.style.margin = '0 5px';
        }
        writerRef.value.appendChild(charContainer);

        const writer = HanziWriter.create(charContainer, char, {
          width: 200,
          height: 200,
          padding: 5,
          showOutline: true,
          strokeColor: '#DC2626', // red-600
          delayBetweenStrokes: 200,
        });
        writerInstances.push(writer);
      }
      
      // Animate sequentially
      for (const writer of writerInstances) {
        await writer.animateCharacter();
      }
    }
  });
};

const currentWordObj = computed(() => words.value[currentIndex.value]);
</script>

<template>
  <div v-if="loading" class="min-h-screen flex items-center justify-center bg-red-50 text-red-800">Loading...</div>

  <!-- Start Screen -->
  <div v-else-if="gameState === 'start'" class="min-h-screen bg-gradient-to-br from-red-50 to-orange-50 flex flex-col items-center justify-center p-4 font-sans">
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
        <div class="w-20 h-20 bg-red-100 rounded-full flex items-center justify-center text-red-600">
          <BookOpen :size="40" />
        </div>
      </div>
      <h1 class="text-3xl font-bold text-gray-800 mb-2">汉字识字卡</h1>
      <p class="text-gray-500 mb-8">学习汉字和句子</p>

      <div class="space-y-4 mb-8 text-left">
        <div>
          <label class="block text-sm font-medium text-gray-700 mb-1">选择课程</label>
          <select 
            v-model="currentSelection"
            @change="handleSelectionChange"
            class="w-full p-3 bg-gray-50 border border-gray-200 rounded-lg focus:ring-2 focus:ring-red-500 outline-none transition-all"
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
        class="w-full py-4 bg-red-600 hover:bg-red-700 text-white rounded-xl font-bold text-lg shadow-lg shadow-red-200 transition-all active:scale-95"
      >
        开始学习
      </button>
    </div>
  </div>

  <!-- Finished Screen -->
  <div v-else-if="gameState === 'finished'" class="min-h-screen bg-red-50 flex flex-col items-center justify-center p-4 font-sans">
    <div class="max-w-md w-full bg-white rounded-2xl shadow-xl p-8 text-center">
       <h2 class="text-3xl font-bold text-gray-800 mb-4">完成!</h2>
       <p class="text-gray-600 mb-8">你已经学习了 {{ words.length }} 个生字.</p>
       
       <button 
          @click="gameState = 'start'"
          class="px-6 py-3 bg-gray-800 text-white rounded-full font-medium hover:bg-gray-900 transition-colors flex items-center gap-2 mx-auto"
        >
          <RefreshCw :size="18" /> 返回菜单
        </button>
    </div>
  </div>

  <!-- Playing Screen -->
  <div v-else class="min-h-screen bg-red-50 flex flex-col items-center justify-center p-4 font-sans">
    <div class="max-w-2xl w-full bg-white rounded-2xl shadow-xl p-8 relative min-h-[500px] flex flex-col">
      <!-- Progress -->
      <div class="absolute top-0 left-0 w-full h-2 bg-gray-100">
        <div 
          class="h-full bg-red-500 transition-all duration-300"
          :style="{ width: `${((currentIndex + 1) / words.length) * 100}%` }"
        />
      </div>

      <div class="mt-4 flex justify-between items-center mb-4">
        <span class="text-gray-400 text-sm">第 {{ currentIndex + 1 }} 个 / 共 {{ words.length }} 个</span>
        <button 
          @click="gameState = 'start'" 
          class="text-sm text-gray-500 hover:text-gray-800 underline"
        >
          退出
        </button>
      </div>

      <!-- Card Content -->
      <div class="flex-1 flex flex-col items-center justify-center">
        <!-- Hanzi Display -->
        <div class="mb-8 text-center relative h-64 flex items-center justify-center">
          <div v-if="!isWriting">
            <h2 class="text-8xl md:text-9xl font-bold text-gray-800 mb-4 font-serif">
              {{ currentWordObj?.word }}
            </h2>
          </div>
          <div v-else ref="writerRef" class="flex flex-wrap justify-center items-center gap-2"></div>
        </div>

        <!-- Hidden/Revealed Info -->
        <div class="transition-opacity duration-500 text-center" :class="{ 'opacity-100': isRevealed, 'opacity-0': !isRevealed }">
          <p class="text-2xl text-red-600 font-medium mb-2">{{ currentWordObj?.definition.split('-')[0] }}</p>
          <p class="text-xl text-gray-600">{{ currentWordObj?.definition.split('-').slice(1).join('-') }}</p>
        </div>
      </div>

      <!-- Controls -->
      <div class="mt-8 flex justify-center items-center gap-6">
        <button v-if="!isRevealed"
          @click="handleReveal"
          class="px-8 py-4 bg-red-600 hover:bg-red-700 text-white rounded-full font-bold text-xl shadow-lg shadow-red-200 transition-all active:scale-95 flex items-center gap-2"
        >
          <Eye :size="24" /> 显示答案
        </button>
        <template v-else>
          <button 
            @click="speakWord(currentWordObj.word)"
            class="w-16 h-16 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center hover:bg-blue-200 transition-transform active:scale-95 shadow-sm"
            title="朗读"
          >
            <Volume2 :size="32" />
          </button>

          <button 
            @click="showStrokeOrder"
            class="w-16 h-16 rounded-full bg-orange-100 text-orange-600 flex items-center justify-center hover:bg-orange-200 transition-transform active:scale-95 shadow-sm"
            title="书写演示"
          >
            <PenTool :size="32" />
          </button>
          
          <button 
            @click="nextWord"
            class="px-8 py-4 bg-green-500 hover:bg-green-600 text-white rounded-full font-bold text-xl shadow-lg shadow-green-200 transition-all active:scale-95 flex items-center gap-2"
          >
            下一个 <ArrowRight :size="24" />
          </button>
        </template>
      </div>
    </div>
  </div>
</template>

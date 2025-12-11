<script setup>
import { ref, onMounted, computed } from 'vue';
import { Volume2, BookOpen, Search, Book, ChevronLeft, ChevronRight, Feather } from 'lucide-vue-next';

const emit = defineEmits(['back']);

const poems = ref([]);
const loading = ref(false);
const structure = ref({});
const lessonOptions = computed(() => {
  const options = [];
  if (structure.value) {
    for (const [book, files] of Object.entries(structure.value)) {
      if (!book.toLowerCase().includes('poetry')) continue;
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
const searchQuery = ref('');
const selectedPoem = ref(null);

onMounted(() => {
  fetchStructure();
});

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

const handleSelectionChange = async () => {
  if (!currentSelection.value) return;
  
  const [book, file] = currentSelection.value.split('|');
  loading.value = true;
  selectedPoem.value = null;
  
  try {
    const res = await fetch(`http://localhost:3003/api/words?book=${book}&file=${file}`);
    const data = await res.json();
    poems.value = data;
    if (poems.value.length > 0) {
      selectedPoem.value = poems.value[0];
    }
  } catch (error) {
    console.error('Failed to fetch poems', error);
  } finally {
    loading.value = false;
  }
};

const filteredPoems = computed(() => {
  if (!searchQuery.value) return poems.value;
  const query = searchQuery.value.toLowerCase();
  return poems.value.filter(poem => 
    poem.title.includes(query) || 
    poem.author.includes(query)
  );
});

const selectPoem = (poem) => {
  selectedPoem.value = poem;
};

const currentIndex = computed(() => {
  if (!selectedPoem.value || !poems.value) return -1;
  return poems.value.findIndex(p => p.title === selectedPoem.value.title);
});

const prevPoem = () => {
  if (currentIndex.value > 0) {
    selectedPoem.value = poems.value[currentIndex.value - 1];
  }
};

const nextPoem = () => {
  if (currentIndex.value < poems.value.length - 1) {
    selectedPoem.value = poems.value[currentIndex.value + 1];
  }
};

const playSound = (poem) => {
  const text = `${poem.title}。${poem.dynasty}，${poem.author}。${poem.content.join('，')}`;
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'zh-CN';
  utterance.rate = 0.8;
  window.speechSynthesis.speak(utterance);
};
</script>

<template>
  <div class="min-h-screen bg-rose-50 font-sans flex flex-col">
    <!-- Header -->
    <header class="bg-white shadow-sm p-4 sticky top-0 z-10">
      <div class="max-w-6xl mx-auto flex items-center justify-between">
        <button 
          @click="$emit('back')"
          class="text-gray-500 hover:text-gray-800 flex items-center gap-2"
        >
          ← 返回菜单
        </button>
        <h1 class="text-xl font-bold text-gray-800 flex items-center gap-2">
          <Feather class="text-rose-600" /> 古诗欣赏
        </h1>
        <div class="w-20"></div>
      </div>
    </header>

    <div class="flex-1 max-w-6xl mx-auto w-full p-4 md:p-8 flex flex-col md:flex-row gap-8">
      
      <!-- Sidebar / List -->
      <div class="w-full md:w-1/3 flex flex-col gap-4">
        <div class="bg-white p-4 rounded-xl shadow-sm">
          <label class="block text-sm font-medium text-gray-700 mb-2">选择章节</label>
          <select 
            v-model="currentSelection"
            @change="handleSelectionChange"
            class="w-full p-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-rose-500"
          >
             <option value="">请选择...</option>
             <option v-for="option in lessonOptions" :key="option.value" :value="option.value">
               {{ option.label }}
             </option>
          </select>
        </div>

        <div class="bg-white p-4 rounded-xl shadow-sm flex-1 flex flex-col min-h-[400px]">
          <div class="relative mb-4">
            <Search class="absolute left-3 top-2.5 text-gray-400" :size="20" />
            <input 
              v-model="searchQuery"
              type="text" 
              placeholder="搜索古诗..." 
              class="w-full pl-10 pr-4 py-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-rose-500"
            />
          </div>

          <div class="flex-1 overflow-y-auto space-y-2 pr-2 custom-scrollbar">
            <div 
              v-for="poem in filteredPoems" 
              :key="poem.title"
              @click="selectPoem(poem)"
              class="p-3 rounded-lg cursor-pointer transition-colors flex justify-between items-center"
              :class="selectedPoem === poem ? 'bg-rose-100 border-l-4 border-rose-500' : 'hover:bg-gray-50 border-l-4 border-transparent'"
            >
              <span class="font-bold text-gray-800">{{ poem.title }}</span>
              <span class="text-xs text-gray-500">{{ poem.author }}</span>
            </div>
            <div v-if="filteredPoems.length === 0" class="text-center text-gray-400 py-8">
              没有找到相关古诗
            </div>
          </div>
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 bg-white rounded-2xl shadow-lg p-8 relative min-h-[500px]">
        <div v-if="selectedPoem" class="animate-fade-in flex flex-col h-full">
          <div class="flex justify-between items-start mb-6 border-b border-gray-100 pb-6">
            <div>
              <h2 class="text-4xl font-bold text-gray-800 mb-2 font-serif">{{ selectedPoem.title }}</h2>
              <p class="text-lg text-rose-600 font-medium">[{{ selectedPoem.dynasty }}] {{ selectedPoem.author }}</p>
            </div>
            <button 
              @click="playSound(selectedPoem)"
              class="w-12 h-12 rounded-full bg-rose-100 text-rose-600 flex items-center justify-center hover:bg-rose-200 transition-transform active:scale-95"
            >
              <Volume2 :size="24" />
            </button>
          </div>

          <!-- Poem Content -->
          <div class="mb-8 flex-1 flex flex-col items-center justify-center py-8">
            <div class="text-center space-y-4">
              <p v-for="(line, idx) in selectedPoem.content" :key="idx" class="text-3xl text-gray-800 font-serif leading-relaxed">
                {{ line }}
              </p>
            </div>
          </div>

          <!-- Explanation -->
          <div class="bg-gray-50 p-6 rounded-xl border-l-4 border-rose-400 mb-8">
            <h3 class="text-sm font-bold text-gray-400 uppercase tracking-wider mb-2">译文与赏析</h3>
            <p class="text-gray-700 leading-relaxed">{{ selectedPoem.explanation }}</p>
          </div>

          <!-- Navigation Buttons -->
          <div class="flex justify-between items-center pt-6 border-t border-gray-100">
            <button 
              @click="prevPoem"
              :disabled="currentIndex <= 0"
              class="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
            >
              <ChevronLeft :size="20" /> 上一首
            </button>
            <span class="text-gray-400 text-sm">{{ currentIndex + 1 }} / {{ poems.length }}</span>
            <button 
              @click="nextPoem"
              :disabled="currentIndex >= poems.length - 1"
              class="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
            >
              下一首 <ChevronRight :size="20" />
            </button>
          </div>
        </div>

        <div v-else class="flex flex-col items-center justify-center h-full text-gray-400">
          <Book :size="64" class="mb-4 text-rose-100" />
          <p class="text-lg">请从左侧选择一首古诗开始阅读</p>
        </div>
      </div>

    </div>
  </div>
</template>

<style scoped>
.custom-scrollbar::-webkit-scrollbar {
  width: 6px;
}
.custom-scrollbar::-webkit-scrollbar-track {
  background: #f1f1f1;
}
.custom-scrollbar::-webkit-scrollbar-thumb {
  background: #fda4af;
  border-radius: 3px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #e11d48;
}
</style>

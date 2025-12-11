<script setup>
import { ref, onMounted, computed } from 'vue';
import { Volume2, BookOpen, Search, Book } from 'lucide-vue-next';

const emit = defineEmits(['back']);

const idioms = ref([]);
const loading = ref(false);
const structure = ref({});
const lessonOptions = computed(() => {
  const options = [];
  if (structure.value) {
    for (const [book, files] of Object.entries(structure.value)) {
      if (!book.toLowerCase().includes('idioms')) continue;
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
const selectedIdiom = ref(null);

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
  selectedIdiom.value = null;
  
  try {
    const res = await fetch(`http://localhost:3003/api/words?book=${book}&file=${file}`);
    const data = await res.json();
    idioms.value = data;
  } catch (error) {
    console.error('Failed to fetch idioms', error);
  } finally {
    loading.value = false;
  }
};

const filteredIdioms = computed(() => {
  if (!searchQuery.value) return idioms.value;
  const query = searchQuery.value.toLowerCase();
  return idioms.value.filter(idiom => 
    idiom.word.includes(query) || 
    (idiom.pinyin && idiom.pinyin.toLowerCase().includes(query))
  );
});

const selectIdiom = (idiom) => {
  selectedIdiom.value = idiom;
};

const playSound = (text) => {
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'zh-CN';
  utterance.rate = 0.8;
  window.speechSynthesis.speak(utterance);
};
</script>

<template>
  <div class="min-h-screen bg-emerald-50 font-sans flex flex-col">
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
          <BookOpen class="text-emerald-600" /> 成语故事
        </h1>
        <div class="w-20"></div> <!-- Spacer -->
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
            class="w-full p-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-emerald-500"
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
              placeholder="搜索成语..." 
              class="w-full pl-10 pr-4 py-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-emerald-500"
            />
          </div>

          <div class="flex-1 overflow-y-auto space-y-2 pr-2 custom-scrollbar">
            <div 
              v-for="idiom in filteredIdioms" 
              :key="idiom.word"
              @click="selectIdiom(idiom)"
              class="p-3 rounded-lg cursor-pointer transition-colors flex justify-between items-center"
              :class="selectedIdiom === idiom ? 'bg-emerald-100 border-l-4 border-emerald-500' : 'hover:bg-gray-50 border-l-4 border-transparent'"
            >
              <span class="font-bold text-gray-800">{{ idiom.word }}</span>
              <span class="text-xs text-gray-500">{{ idiom.pinyin }}</span>
            </div>
            <div v-if="filteredIdioms.length === 0" class="text-center text-gray-400 py-8">
              没有找到相关成语
            </div>
          </div>
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 bg-white rounded-2xl shadow-lg p-8 relative min-h-[500px]">
        <div v-if="selectedIdiom" class="animate-fade-in">
          <div class="flex justify-between items-start mb-6 border-b border-gray-100 pb-6">
            <div>
              <h2 class="text-5xl font-bold text-gray-800 mb-2 font-serif">{{ selectedIdiom.word }}</h2>
              <p class="text-xl text-emerald-600 font-medium">{{ selectedIdiom.pinyin }}</p>
            </div>
            <button 
              @click="playSound(selectedIdiom.word)"
              class="w-12 h-12 rounded-full bg-emerald-100 text-emerald-600 flex items-center justify-center hover:bg-emerald-200 transition-transform active:scale-95"
            >
              <Volume2 :size="24" />
            </button>
          </div>

          <div class="mb-8">
            <h3 class="text-sm font-bold text-gray-400 uppercase tracking-wider mb-2">释义 (Definition)</h3>
            <p class="text-lg text-gray-700 leading-relaxed bg-gray-50 p-4 rounded-lg border-l-4 border-emerald-400">
              {{ selectedIdiom.definition }}
            </p>
          </div>

          <div class="mb-8">
            <h3 class="text-sm font-bold text-gray-400 uppercase tracking-wider mb-2">典故 (Story)</h3>
            <div class="prose max-w-none text-gray-600 leading-relaxed">
              <p>{{ selectedIdiom.story }}</p>
            </div>
          </div>
          
          <div v-if="selectedIdiom.origin" class="text-right text-sm text-gray-400 italic">
            —— 出处：{{ selectedIdiom.origin }}
          </div>
        </div>

        <div v-else class="flex flex-col items-center justify-center h-full text-gray-400">
          <Book :size="64" class="mb-4 text-emerald-100" />
          <p class="text-lg">请从左侧选择一个成语开始阅读</p>
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
  background: #d1fae5;
  border-radius: 3px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #10b981;
}
</style>

<script setup>
import { ref, onMounted, computed } from 'vue';
import { Volume2, BookOpen, Search, Book, ChevronLeft, ChevronRight, Scroll } from 'lucide-vue-next';

const emit = defineEmits(['back']);

const articles = ref([]);
const loading = ref(false);
const structure = ref({});
const lessonOptions = computed(() => {
  const options = [];
  if (structure.value) {
    for (const [book, files] of Object.entries(structure.value)) {
      if (!book.toLowerCase().includes('classical')) continue;
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
const selectedArticle = ref(null);

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
  selectedArticle.value = null;
  
  try {
    const res = await fetch(`http://localhost:3003/api/words?book=${book}&file=${file}`);
    const data = await res.json();
    articles.value = data;
    if (articles.value.length > 0) {
      selectedArticle.value = articles.value[0];
    }
  } catch (error) {
    console.error('Failed to fetch articles', error);
  } finally {
    loading.value = false;
  }
};

const filteredArticles = computed(() => {
  if (!searchQuery.value) return articles.value;
  const query = searchQuery.value.toLowerCase();
  return articles.value.filter(article => 
    article.title.includes(query) || 
    article.author.includes(query)
  );
});

const selectArticle = (article) => {
  selectedArticle.value = article;
};

const currentIndex = computed(() => {
  if (!selectedArticle.value || !articles.value) return -1;
  return articles.value.findIndex(a => a.title === selectedArticle.value.title);
});

const prevArticle = () => {
  if (currentIndex.value > 0) {
    selectedArticle.value = articles.value[currentIndex.value - 1];
  }
};

const nextArticle = () => {
  if (currentIndex.value < articles.value.length - 1) {
    selectedArticle.value = articles.value[currentIndex.value + 1];
  }
};

const playSound = (article) => {
  const text = `${article.title}。${article.dynasty}，${article.author}。${article.content.join(' ')}`;
  const utterance = new SpeechSynthesisUtterance(text);
  utterance.lang = 'zh-CN';
  utterance.rate = 0.8;
  window.speechSynthesis.speak(utterance);
};
</script>

<template>
  <div class="min-h-screen bg-stone-50 font-sans flex flex-col">
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
          <Scroll class="text-stone-600" /> 古文阅读
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
            class="w-full p-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-stone-500"
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
              placeholder="搜索古文..." 
              class="w-full pl-10 pr-4 py-2 bg-gray-50 border border-gray-200 rounded-lg outline-none focus:ring-2 focus:ring-stone-500"
            />
          </div>

          <div class="flex-1 overflow-y-auto space-y-2 pr-2 custom-scrollbar">
            <div 
              v-for="article in filteredArticles" 
              :key="article.title"
              @click="selectArticle(article)"
              class="p-3 rounded-lg cursor-pointer transition-colors flex justify-between items-center"
              :class="selectedArticle === article ? 'bg-stone-100 border-l-4 border-stone-500' : 'hover:bg-gray-50 border-l-4 border-transparent'"
            >
              <span class="font-bold text-gray-800">{{ article.title }}</span>
              <span class="text-xs text-gray-500">{{ article.author }}</span>
            </div>
            <div v-if="filteredArticles.length === 0" class="text-center text-gray-400 py-8">
              没有找到相关古文
            </div>
          </div>
        </div>
      </div>

      <!-- Main Content -->
      <div class="flex-1 bg-white rounded-2xl shadow-lg p-8 relative min-h-[500px]">
        <div v-if="selectedArticle" class="animate-fade-in flex flex-col h-full">
          <div class="flex justify-between items-start mb-6 border-b border-gray-100 pb-6">
            <div>
              <h2 class="text-4xl font-bold text-gray-800 mb-2 font-serif">{{ selectedArticle.title }}</h2>
              <p class="text-lg text-stone-600 font-medium">[{{ selectedArticle.dynasty }}] {{ selectedArticle.author }}</p>
            </div>
            <button 
              @click="playSound(selectedArticle)"
              class="w-12 h-12 rounded-full bg-stone-100 text-stone-600 flex items-center justify-center hover:bg-stone-200 transition-transform active:scale-95"
            >
              <Volume2 :size="24" />
            </button>
          </div>

          <!-- Content -->
          <div class="mb-8 flex-1 overflow-y-auto max-h-[400px] custom-scrollbar pr-4">
             <div class="prose max-w-none">
              <p v-for="(para, idx) in selectedArticle.content" :key="idx" class="text-xl text-gray-800 font-serif leading-loose mb-4 indent-8">
                {{ para }}
              </p>
            </div>
          </div>

          <!-- Explanation -->
          <div class="bg-gray-50 p-6 rounded-xl border-l-4 border-stone-400 mb-8">
            <h3 class="text-sm font-bold text-gray-400 uppercase tracking-wider mb-2">译文与赏析</h3>
            <p class="text-gray-700 leading-relaxed">{{ selectedArticle.explanation }}</p>
          </div>

          <!-- Navigation Buttons -->
          <div class="flex justify-between items-center pt-6 border-t border-gray-100">
            <button 
              @click="prevArticle"
              :disabled="currentIndex <= 0"
              class="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
            >
              <ChevronLeft :size="20" /> 上一篇
            </button>
            <span class="text-gray-400 text-sm">{{ currentIndex + 1 }} / {{ articles.length }}</span>
            <button 
              @click="nextArticle"
              :disabled="currentIndex >= articles.length - 1"
              class="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
            >
              下一篇 <ChevronRight :size="20" />
            </button>
          </div>
        </div>

        <div v-else class="flex flex-col items-center justify-center h-full text-gray-400">
          <Book :size="64" class="mb-4 text-stone-100" />
          <p class="text-lg">请从左侧选择一篇古文开始阅读</p>
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
  background: #d6d3d1;
  border-radius: 3px;
}
.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: #a8a29e;
}
</style>

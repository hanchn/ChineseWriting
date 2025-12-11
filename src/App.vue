<script setup>
import { ref } from 'vue';
import { BookOpen, PenTool, Table, Music, Book } from 'lucide-vue-next';
import HanziGame from './components/HanziGame.vue';
import SentenceGame from './components/SentenceGame.vue';
import PinyinGame from './components/PinyinGame.vue';
import IdiomGame from './components/IdiomGame.vue';

const currentApp = ref('home'); // home, hanzi, sentences, pinyin, idioms
</script>

<template>
  <div class="min-h-screen bg-slate-50 font-sans">
    <HanziGame v-if="currentApp === 'hanzi'" @back="currentApp = 'home'" />
    <SentenceGame v-else-if="currentApp === 'sentences'" @back="currentApp = 'home'" />
    <PinyinGame v-else-if="currentApp === 'pinyin'" @back="currentApp = 'home'" />
    <IdiomGame v-else-if="currentApp === 'idioms'" @back="currentApp = 'home'" />

    <div v-else class="max-w-6xl mx-auto p-8">
      <header class="text-center mb-16 pt-8">
        <h1 class="text-4xl md:text-5xl font-bold text-gray-800 mb-4">
          中文学习中心
        </h1>
        <p class="text-xl text-gray-500">选择你的学习路径</p>
      </header>

      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
        <!-- Chinese Section -->
        <div class="col-span-full mb-4">
          <h2 class="text-2xl font-bold text-gray-700 flex items-center gap-2">
            <PenTool class="text-red-600" /> 中文学习
          </h2>
        </div>

        <!-- Entry Point 1: Hanzi Recognition -->
        <div 
          @click="currentApp = 'hanzi'"
          class="bg-white p-8 rounded-2xl shadow-lg hover:shadow-xl transition-all cursor-pointer border-2 border-transparent hover:border-red-500 group"
        >
          <div class="w-16 h-16 bg-red-100 rounded-2xl flex items-center justify-center text-red-600 mb-6 group-hover:scale-110 transition-transform">
            <BookOpen :size="32" />
          </div>
          <h3 class="text-2xl font-bold text-gray-800 mb-2">汉字识字卡</h3>
          <p class="text-gray-500">识字、读拼音、学习释义</p>
        </div>

        <!-- Entry Point 2: Sentence Reading -->
        <div 
          @click="currentApp = 'sentences'"
          class="bg-white p-8 rounded-2xl shadow-lg hover:shadow-xl transition-all cursor-pointer border-2 border-transparent hover:border-amber-500 group"
        >
          <div class="w-16 h-16 bg-amber-100 rounded-2xl flex items-center justify-center text-amber-600 mb-6 group-hover:scale-110 transition-transform">
            <Book :size="32" />
          </div>
          <h3 class="text-2xl font-bold text-gray-800 mb-2">句子阅读</h3>
          <p class="text-gray-500">在语境中练习阅读完整句子</p>
        </div>

        <div 
          @click="currentApp = 'pinyin'"
          class="bg-white p-8 rounded-2xl shadow-lg hover:shadow-xl transition-all cursor-pointer border-2 border-transparent hover:border-indigo-500 group"
        >
          <div class="w-16 h-16 bg-indigo-100 rounded-2xl flex items-center justify-center text-indigo-600 mb-6 group-hover:scale-110 transition-transform">
            <Table :size="32" />
          </div>
          <h3 class="text-2xl font-bold text-gray-800 mb-2">拼音表</h3>
          <p class="text-gray-500">交互式带发音拼音表</p>
        </div>

        <div 
          @click="currentApp = 'idioms'"
          class="bg-white p-8 rounded-2xl shadow-lg hover:shadow-xl transition-all cursor-pointer border-2 border-transparent hover:border-emerald-500 group"
        >
          <div class="w-16 h-16 bg-emerald-100 rounded-2xl flex items-center justify-center text-emerald-600 mb-6 group-hover:scale-110 transition-transform">
            <Music :size="32" />
          </div>
          <h3 class="text-2xl font-bold text-gray-800 mb-2">成语故事</h3>
          <p class="text-gray-500">通过成语故事学习中国文化</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const supabase = useSupabaseClient()
const lastPingTime = ref(null)
const status = ref('waiting') // 'waiting', 'pinging', 'success', 'error'
const errorMsg = ref('')
let intervalId = null

// 輕量級喚醒函數
const pingDatabase = async () => {
  status.value = 'pinging'
  errorMsg.value = ''
  
  try {
    // 故意發送一個最輕量的請求，只抓一筆資料的 ID，不消耗效能
    const { error } = await supabase
      .from('counseling_logs')
      .select('id')
      .limit(1)

    if (error) throw error

    status.value = 'success'
    lastPingTime.value = new Date().toLocaleString('zh-TW', { 
      year: 'numeric', month: '2-digit', day: '2-digit',
      hour: '2-digit', minute: '2-digit', second: '2-digit'
    })
  } catch (err) {
    status.value = 'error'
    errorMsg.value = err.message || '連線失敗'
    console.error('Ping 失敗:', err)
  }
}

onMounted(() => {
  // 網頁一載入，立刻敲門一次
  pingDatabase()

  // 設定定時器：每 15 分鐘 (900,000 毫秒) 自動敲門一次
  intervalId = setInterval(() => {
    pingDatabase()
  }, 15 * 60 * 1000)
})

onUnmounted(() => {
  // 離開網頁時清除定時器
  if (intervalId) clearInterval(intervalId)
})
</script>

<template>
  <div class="min-h-[100dvh] flex flex-col items-center justify-center bg-gray-900 text-gray-100 p-4 font-sans">
    
    <div class="w-full max-w-sm bg-gray-800 rounded-2xl shadow-2xl p-6 border border-gray-700">
      
      <div class="text-center mb-6">
        <h1 class="text-2xl font-bold tracking-wider mb-2 flex items-center justify-center gap-2">
          <span>⚡</span> 資料庫保活系統
        </h1>
        <p class="text-xs text-gray-400">
          Supabase Keep-Alive Service
        </p>
      </div>

      <div class="bg-gray-900 rounded-xl p-5 mb-6 border border-gray-700 shadow-inner text-center">
        <div class="text-xs text-gray-500 mb-2 uppercase tracking-widest">最後喚醒時間</div>
        <div class="text-xl font-mono" :class="status === 'success' ? 'text-green-400' : 'text-gray-400'">
          {{ lastPingTime || '-- / -- / --  --:--:--' }}
        </div>
      </div>

      <div class="flex items-center justify-center gap-2 mb-6 h-6 text-sm font-bold">
        <span v-if="status === 'pinging'" class="text-yellow-400 animate-pulse">
          🔄 正在敲擊資料庫...
        </span>
        <span v-else-if="status === 'success'" class="text-green-400">
          ✅ 訊號已確認，資料庫活躍中
        </span>
        <span v-else-if="status === 'error'" class="text-red-400">
          ❌ 喚醒失敗
        </span>
      </div>

      <div v-if="errorMsg" class="bg-red-900/30 text-red-400 text-xs p-3 rounded-lg mb-6 break-words border border-red-800/50">
        錯誤細節: {{ errorMsg }}
      </div>

      <button
        @click="pingDatabase"
        :disabled="status === 'pinging'"
        class="w-full py-3.5 bg-blue-600 hover:bg-blue-500 text-white rounded-xl font-bold transition-colors disabled:opacity-50 disabled:cursor-not-allowed shadow-lg"
      >
        手動發送 Ping 訊號
      </button>

      <div class="mt-4 text-center">
        <NuxtLink to="/" class="text-xs text-gray-500 hover:text-gray-300 underline underline-offset-4">
          返回首頁
        </NuxtLink>
      </div>

    </div>

    <p class="mt-6 text-xs text-gray-600 text-center max-w-xs">
      保持此分頁開啟，系統將每 15 分鐘自動在背景喚醒資料庫一次。
    </p>
  </div>
</template>

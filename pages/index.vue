<script setup>
import { ref, watch, onMounted } from 'vue'
import { useDebounceFn } from '@vueuse/core'
import { useRouter } from 'vue-router'

const supabase = useSupabaseClient()
const router = useRouter()

const content = ref('')
const isSaving = ref(false)
const lastSavedTime = ref(null)

// 產生這次輸入專屬的 ID (重新整理就會產生新的)
const sessionId = ref('')
const today = new Date().toISOString().split('T')[0]

onMounted(() => {
  sessionId.value = crypto.randomUUID()
})

// 自動存檔邏輯
const saveToSupabase = async (newContent) => {
  if (!newContent || !newContent.trim()) return

  isSaving.value = true
  
  await supabase.from('counseling_logs').upsert({ 
    id: sessionId.value, 
    record_date: today,
    content: newContent,
    updated_at: new Date().toISOString()
  })
  
  isSaving.value = false
  lastSavedTime.value = new Date().toLocaleTimeString('zh-TW', { hour: '2-digit', minute: '2-digit' })
}

const autoSave = useDebounceFn((newVal) => {
  saveToSupabase(newVal)
}, 1500)

watch(content, (newVal) => {
  autoSave(newVal)
})

const copyContent = async () => {
  try {
    await navigator.clipboard.writeText(content.value)
    alert('✅ 文字已複製！')
  } catch (err) {
    console.error('複製失敗', err)
  }
}

const startNewRecord = () => {
  content.value = ''
  sessionId.value = crypto.randomUUID()
  lastSavedTime.value = null
}

// ==========================================
// 🌟 更新：動態密碼驗證與瀏覽器記憶功能
// ==========================================
const goToHistory = () => {
  const d = new Date()
  
  // 取得西元年後兩碼、月、日並補零
  const yy = String(d.getFullYear()).slice(-2)
  const mm = String(d.getMonth() + 1).padStart(2, '0')
  const dd = String(d.getDate()).padStart(2, '0')
  
  // 當日日期字串 (例如 "260917")
  const dateStr = `${yy}${mm}${dd}`
  // 組合當日動態密碼 (例如 "26091759")
  const dynamicPassword = `${dateStr}59`

  // 1. 檢查瀏覽器是否已經記錄了「今天」的驗證
  const savedAuthDate = localStorage.getItem('history_auth_date')
  
  if (savedAuthDate === dateStr) {
    // 若記錄的日期就是今天，免密碼直接放行
    router.push('/history')
    return
  }

  // 2. 如果今天還沒驗證過，彈出輸入框
  const userInput = prompt('請輸入密碼：')
  
  if (userInput === dynamicPassword) {
    // 密碼正確，把「今天日期」存進瀏覽器，然後跳轉
    localStorage.setItem('history_auth_date', dateStr)
    router.push('/history')
  } else if (userInput !== null) { 
    // 若使用者輸入錯誤 (且不是按取消)
    alert('密碼錯誤')
  }
}
</script>

<template>
  <div class="flex flex-col min-h-[100dvh] p-4 max-w-2xl mx-auto bg-gray-100">
    
    <div class="flex justify-between items-end mb-4 px-1">
      <div>
        <h1 class="text-2xl font-bold text-gray-800 tracking-wide mb-1">新增輔導日誌</h1>
        
        <button 
          @click="goToHistory" 
          class="text-sm bg-blue-100 text-blue-700 px-3 py-1.5 rounded-lg font-bold shadow-sm active:bg-blue-200 transition-colors"
        >
          📂 前往歷史紀錄
        </button>

      </div>
      <span class="text-xs font-medium text-gray-400 mb-1">
        {{ isSaving ? '儲存中...' : (lastSavedTime ? `${lastSavedTime} 已存檔` : '') }}
      </span>
    </div>

    <textarea 
      v-model="content" 
      class="flex-none h-56 p-4 border rounded-2xl text-[16px] leading-relaxed resize-none shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 bg-white"
      placeholder="點擊此處，用手機鍵盤麥克風口述新事件..."
    ></textarea>
    
    <div class="mt-3 mb-2 flex gap-2">
      <button @click="startNewRecord" class="px-4 py-3 bg-gray-300 text-gray-700 rounded-xl font-bold active:bg-gray-400">
        新增下一筆
      </button>
      <button @click="copyContent" class="flex-1 py-3 bg-gray-800 text-white rounded-xl font-bold shadow-md active:bg-gray-700">
        複製文字內容
      </button>
    </div>

    <EvidenceUploader :currentDate="today" />
    
    <!-- 🌟 新增：資料庫測試小入口 (放到底部) -->
    <div class="mt-8 text-center pb-4">
      <NuxtLink 
        to="/test-db" 
        class="text-xs text-gray-400 hover:text-blue-500 transition-colors flex items-center justify-center gap-1"
      >
        <span>🔧</span> 資料庫連線檢測
      </NuxtLink>
    </div>

  </div>
</template>

<style scoped>
/* 避免 iOS Safari 在輸入時畫面亂跳 */
textarea {
  font-size: 16px !important;
}
</style>

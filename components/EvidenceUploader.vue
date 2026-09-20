<script setup>
import { ref, onMounted, computed } from 'vue'

const props = defineProps({
  currentDate: { type: String, required: true }
})

const supabase = useSupabaseClient()
const config = useRuntimeConfig()
const fileInput = ref(null)
const isUploading = ref(false) // false, true, 'processing', 'retrying'
const uploadProgress = ref(0)
const evidenceList = ref([])

// 🌟 設定伺服器清單 (自動讀取環境變數)
const apiUrls = computed(() => {
  const url1 = config.public.hfApiUrl ? config.public.hfApiUrl.replace(/\/$/, '') : ''
  // 假設第二個 API 網址放在 hfApiUrl2
  const url2 = config.public.hfApiUrl2 ? config.public.hfApiUrl2.replace(/\/$/, '') : ''
  return [url1, url2].filter(url => url !== '')
})

// 綁定使用者目前選擇的優先伺服器 (0 或 1)
const selectedApiIndex = ref(0)

const fetchEvidence = async () => {
  const { data } = await supabase
    .from('evidence_logs')
    .select('*')
    .eq('log_date', props.currentDate)
  if (data) evidenceList.value = data
}

onMounted(() => { fetchEvidence() })

const formatRecordingTime = (timestamp) => {
  const d = new Date(timestamp)
  const pad = (n) => String(n).padStart(2, '0')
  return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())}_${pad(d.getHours())}${pad(d.getMinutes())}`
}

const getDownloadUrl = (messageId) => {
  const currentApi = apiUrls.value[selectedApiIndex.value] || apiUrls.value[0] || '#'
  return `${currentApi}/download/${messageId}`
}

// ==========================================
// 🌟 核心模組：負責對「單一指定的 API」執行完整上傳與輪詢
// ==========================================
const uploadToServer = async (file, apiUrl, newFilename) => {
  const formData = new FormData()
  formData.append('file', file, newFilename)

  uploadProgress.value = 0
  
  // 階段 1：上傳至指定的 Render 伺服器
  const taskId = await new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest()
    xhr.open('POST', `${apiUrl}/upload/`)
    
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) {
        uploadProgress.value = Math.floor((e.loaded / e.total) * 100)
      }
    }

    xhr.onload = () => {
      if (xhr.status >= 200 && xhr.status < 300) {
        try {
          const result = JSON.parse(xhr.responseText)
          if (result.success) resolve(result.task_id)
          else reject(new Error('伺服器拒絕產生任務'))
        } catch (err) {
          reject(new Error('伺服器回傳格式錯誤'))
        }
      } else {
        reject(new Error(`上傳 Render 失敗: 狀態碼 ${xhr.status}`))
      }
    }
    xhr.onerror = () => reject(new Error('網路連線中斷'))
    xhr.send(formData)
  })

  // 階段 2：前端安全輪詢
  let taskFinished = false
  while (!taskFinished) {
    try {
      const res = await fetch(`${apiUrl}/status/${taskId}`)
      const data = await res.json()

      if (data.status === 'completed') {
        taskFinished = true 
        return data // 成功，回傳 Telegram 資料
      } else if (data.status === 'failed') {
        taskFinished = true 
        throw new Error('伺服器轉傳 Telegram 失敗')
      } else {
        await new Promise(resolve => setTimeout(resolve, 3000))
      }
    } catch (err) {
      if (err.message === '伺服器轉傳 Telegram 失敗') throw err
      await new Promise(resolve => setTimeout(resolve, 3000))
    }
  }
}

// ==========================================
// 🌟 批次控制模組：處理檔案迴圈、失敗備援切換、寫入資料庫
// ==========================================
const handleBatchUpload = async (event) => {
  if (isUploading.value !== false) {
    console.warn('上傳程序進行中，阻擋重複觸發')
    return
  }

  const files = event.target?.files || fileInput.value?.files
  if (!files || files.length === 0) return

  if (apiUrls.value.length === 0) {
    alert('未設定任何 API 網址！')
    return
  }

  let successCount = 0
  const safeInsertLock = new Set()

  try {
    for (let i = 0; i < files.length; i++) {
      const file = files[i]
      const recordTime = formatRecordingTime(file.lastModified)
      const newFilename = `${recordTime}_${file.name}`
      
      // 決定主線路與備援線路
      const primaryApi = apiUrls.value[selectedApiIndex.value]
      const backupApi = apiUrls.value[selectedApiIndex.value === 0 ? 1 : 0] || primaryApi
      
      let tgData = null

      // 🥊 第一次嘗試：使用使用者選擇的主線路
      try {
        isUploading.value = true
        tgData = await uploadToServer(file, primaryApi, newFilename)
      } catch (err) {
        console.warn(`⚠️ 主線路 (${primaryApi}) 失敗，啟動備援機制...`, err)
        
        // 🥊 第二次嘗試：若主線路失敗，且有設定第二台伺服器，自動切換
        if (primaryApi !== backupApi) {
          try {
            isUploading.value = 'retrying' // 顯示重試狀態
            tgData = await uploadToServer(file, backupApi, newFilename)
          } catch (backupErr) {
            console.error(`❌ 備援線路 (${backupApi}) 也失敗:`, backupErr)
            throw new Error(`主線路與備援線路皆上傳失敗 (${file.name})`)
          }
        } else {
          // 沒有備用線路可以切換，直接報錯
          throw err 
        }
      }

      // ✅ 兩條線路其中一條成功了，寫入資料庫
      isUploading.value = 'processing'
      if (tgData && !safeInsertLock.has(tgData.message_id)) {
        safeInsertLock.add(tgData.message_id)
        
        const { error } = await supabase.from('evidence_logs').insert({
          log_date: props.currentDate,
          title: newFilename.split('.')[0], 
          telegram_url: tgData.telegram_link,
          file_name: newFilename,
          message_id: tgData.message_id 
        })
        
        if (error) console.error('Supabase 寫入失敗:', error)
        else successCount++
      }
    } // 結束 for 迴圈

    if (successCount > 0) {
      alert(`✅ 成功上傳並歸檔 ${successCount} 筆證據！`)
      fetchEvidence()
    }

  } catch (err) {
    alert('上傳發生錯誤：\n' + err.message)
    console.error(err)
  } finally {
    isUploading.value = false
    uploadProgress.value = 0
    if (fileInput.value) fileInput.value.value = null
  }
}

const deleteEvidence = async (id) => {
  if (!confirm('⚠️ 確定要刪除這筆錄音證據的連結嗎？')) return
  const { error } = await supabase.from('evidence_logs').delete().eq('id', id)
  if (!error) evidenceList.value = evidenceList.value.filter(item => item.id !== id)
}
</script>

<template>
  <div class="mt-4 bg-white p-4 rounded-xl shadow-sm border border-gray-200">
    <div class="flex justify-between items-center mb-3">
      <h2 class="text-md font-bold text-gray-800 flex items-center gap-2">
        <span>🎙️</span> 語音證據清單
      </h2>
    </div>

    <!-- 🌟 新增：伺服器選擇器 -->
    <div class="mb-3 p-3 bg-blue-50 rounded-lg border border-blue-100">
      <label class="block text-xs font-bold text-blue-800 mb-1">連線設定 (具備自動備援)</label>
      <select 
        v-model="selectedApiIndex" 
        :disabled="isUploading !== false"
        class="w-full p-2 rounded border border-blue-200 text-sm focus:outline-none focus:ring-2 focus:ring-blue-400 bg-white"
      >
        <option :value="0">🌐 優先使用：伺服器 A (主機)</option>
        <option :value="1" :disabled="apiUrls.length < 2">
          {{ apiUrls.length < 2 ? '⚠️ 尚未設定伺服器 B' : '🌐 優先使用：伺服器 B (備用)' }}
        </option>
      </select>
    </div>

    <input 
      ref="fileInput" 
      type="file" 
      multiple 
      accept="audio/*,video/mp4" 
      class="hidden" 
      @click="$event.target.value = null" 
      @change="handleBatchUpload" 
    />

    <button 
      @click="$refs.fileInput.click()"
      :disabled="isUploading !== false"
      class="w-full py-3 mb-4 border-2 border-dashed border-blue-300 rounded-xl text-blue-600 font-medium hover:bg-blue-50 active:bg-blue-100 transition-colors disabled:opacity-50 disabled:cursor-not-allowed"
    >
      <span v-if="isUploading === false">＋ 點擊選取手機錄音補傳 (可批次多選)</span>
      <span v-else-if="isUploading === true">正在傳送至優先伺服器 ({{ uploadProgress }}%)...</span>
      <span v-else-if="isUploading === 'retrying'">⚠️ 切換至備用伺服器上傳 ({{ uploadProgress }}%)...</span>
      <span v-else-if="isUploading === 'processing'">☁️ 檔案已抵達，後台正轉存至 Telegram...</span>
    </button>

    <ul class="space-y-3">
      <li v-for="item in evidenceList" :key="item.id" class="flex flex-col bg-gray-50 p-3 rounded-lg border border-gray-200 gap-3">
        <span class="text-sm font-bold text-gray-800 break-all">{{ item.title }}</span>
        
        <div class="flex flex-wrap gap-2">
          <a 
            v-if="item.message_id"
            :href="getDownloadUrl(item.message_id)" 
            target="_blank" 
            class="flex-1 text-center px-2 py-2 bg-green-500 text-white rounded-lg text-xs font-bold hover:bg-green-600 shadow-sm whitespace-nowrap"
          >
            ▶️ 立即串流下載🔗
          </a>
          <a 
            :href="item.telegram_url" 
            target="_blank" 
            class="flex-1 text-center px-2 py-2 bg-blue-500 text-white rounded-lg text-xs font-bold hover:bg-blue-600 shadow-sm whitespace-nowrap"
          >
            ✈️ TG 原文
          </a>
          <button 
            @click="deleteEvidence(item.id)" 
            class="px-3 py-2 bg-red-100 text-red-600 rounded-lg text-xs font-bold hover:bg-red-200 shadow-sm whitespace-nowrap"
          >
            🗑️ 刪除
          </button>
        </div>
      </li>
    </ul>
    
    <p v-if="evidenceList.length === 0" class="text-xs text-gray-400 text-center py-2">本日尚無附加錄音證據</p>
  </div>
</template>

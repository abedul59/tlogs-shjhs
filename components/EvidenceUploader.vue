<script setup>
import { ref, onMounted } from 'vue'

const props = defineProps({
  currentDate: { type: String, required: true }
})

const supabase = useSupabaseClient()
const config = useRuntimeConfig()
const fileInput = ref(null)
const isUploading = ref(false) // false, true, 或 'processing'
const uploadProgress = ref(0)
const evidenceList = ref([])

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
  if (!config.public.hfApiUrl) return '#'
  const baseUrl = config.public.hfApiUrl.endsWith('/') 
    ? config.public.hfApiUrl.slice(0, -1) 
    : config.public.hfApiUrl
  return `${baseUrl}/download/${messageId}`
}

const handleBatchUpload = async (event) => {
  // 🛡️ 第一道鎖：如果已經在執行中，嚴格阻擋任何重複點擊或事件觸發
  if (isUploading.value !== false) {
    console.warn('上傳程序進行中，阻擋重複觸發')
    return
  }

  // 確保拿到檔案 (支援 event 或 ref)
  const files = event.target?.files || fileInput.value?.files
  if (!files || files.length === 0) return

  isUploading.value = true
  uploadProgress.value = 0
  let successCount = 0

  // 🛡️ 第二道鎖：記憶已經成功寫入資料庫的任務 ID
  const safeInsertLock = new Set()

  try {
    const rawApiUrl = config.public.hfApiUrl
    if (!rawApiUrl) throw new Error('未設定 API 網址')
    const hfApiUrl = rawApiUrl.replace(/\/$/, '')

    // 依序處理每一個檔案 (避免 Render 記憶體一次被塞爆)
    for (let i = 0; i < files.length; i++) {
      const file = files[i]
      const recordTime = formatRecordingTime(file.lastModified)
      const newFilename = `${recordTime}_${file.name}`

      const formData = new FormData()
      formData.append('file', file, newFilename)

      isUploading.value = true
      
      // 階段 1：上傳到 Render
      const taskId = await new Promise((resolve, reject) => {
        const xhr = new XMLHttpRequest()
        xhr.open('POST', `${hfApiUrl}/upload/`)
        
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
      isUploading.value = 'processing' 
      let taskFinished = false
      
      while (!taskFinished) {
        try {
          const res = await fetch(`${hfApiUrl}/status/${taskId}`)
          const data = await res.json()

          if (data.status === 'completed') {
            taskFinished = true 
            
            // 🛡️ 終極防護：檢查這個任務是否已經存進資料庫過了？
            if (!safeInsertLock.has(taskId)) {
              safeInsertLock.add(taskId) // 立即上鎖
              
              // 確保沒寫入過，才真正呼叫 Supabase
              const { error } = await supabase.from('evidence_logs').insert({
                log_date: props.currentDate,
                title: newFilename.split('.')[0], 
                telegram_url: data.telegram_link,
                file_name: newFilename,
                message_id: data.message_id 
              })
              
              if (error) console.error('Supabase 寫入失敗:', error)
              else successCount++
            }
            
          } else if (data.status === 'failed') {
            taskFinished = true 
            throw new Error('伺服器轉傳 Telegram 失敗')
            
          } else {
            // 尚未完成，乖乖等 3 秒再問
            await new Promise(resolve => setTimeout(resolve, 3000))
          }
        } catch (err) {
          if (err.message === '伺服器轉傳 Telegram 失敗') throw err
          await new Promise(resolve => setTimeout(resolve, 3000))
        }
      }
    }

    if (successCount > 0) {
      alert(`✅ 成功上傳並歸檔 ${successCount} 筆證據！`)
      fetchEvidence()
    } else {
      alert('上傳發生問題，請檢查網路連線。')
    }

  } catch (err) {
    alert('上傳發生錯誤：\n' + err.message)
    console.error(err)
  } finally {
    isUploading.value = false
    uploadProgress.value = 0
    // 清空選取，確保不會觸發第二次
    if (fileInput.value) fileInput.value.value = ''
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

    <!-- 🌟 加入了明確的 onClick 清空機制 -->
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
      <span v-else-if="isUploading === true">正在傳送至伺服器 ({{ uploadProgress }}%)...</span>
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

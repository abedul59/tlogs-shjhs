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

// 專門用來產生乾淨下載網址的函數 (避開 Vue 模板編譯錯誤)
const getDownloadUrl = (messageId) => {
  if (!config.public.hfApiUrl) return '#'
  
  const baseUrl = config.public.hfApiUrl.endsWith('/') 
    ? config.public.hfApiUrl.slice(0, -1) 
    : config.public.hfApiUrl
    
  return `${baseUrl}/download/${messageId}`
}

const handleBatchUpload = async () => {
  const files = fileInput.value?.files
  if (!files || files.length === 0) return

  isUploading.value = true
  uploadProgress.value = 0
  let successCount = 0

  try {
    const rawApiUrl = config.public.hfApiUrl
    if (!rawApiUrl) throw new Error('未設定 API 網址')
    // 在 script 區塊使用正規表達式是安全的
    const hfApiUrl = rawApiUrl.replace(/\/$/, '')

    for (let i = 0; i < files.length; i++) {
      const file = files[i]
      const recordTime = formatRecordingTime(file.lastModified)
      const newFilename = `${recordTime}_${file.name}`

      const formData = new FormData()
      formData.append('file', file, newFilename)

      // ==========================================
      // 階段 1：將檔案上傳至 Render 暫存區，取得 task_id
      // ==========================================
      isUploading.value = true
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

      // ==========================================
      // 階段 2：前端顯示「轉傳中」，並開始輪詢任務進度
      // ==========================================
      isUploading.value = 'processing' 
      
      await new Promise((resolve, reject) => {
        const checkStatus = setInterval(async () => {
          try {
            const res = await fetch(`${hfApiUrl}/status/${taskId}`)
            const data = await res.json()

            if (data.status === 'completed') {
              clearInterval(checkStatus)
              
              // 成功拿到 TG 的資料，正式寫入 Supabase 資料庫
              const { error } = await supabase.from('evidence_logs').insert({
                log_date: props.currentDate,
                title: newFilename.split('.')[0], 
                telegram_url: data.telegram_link,
                file_name: newFilename,
                message_id: data.message_id 
              })
              
              if (!error) successCount++
              resolve()
            } else if (data.status === 'failed') {
              clearInterval(checkStatus)
              reject(new Error('伺服器轉傳 Telegram 失敗'))
            }
            // 若狀態仍是 processing 或 pending，則不作處理，等待下一次詢問
          } catch (err) {
            console.error('輪詢失敗，將於 3 秒後自動重試', err)
          }
        }, 3000) // 每 3 秒發送一次查詢
      })
    }

    if (successCount > 0) {
      alert(`✅ 成功上傳並歸檔 ${successCount} 筆證據！`)
      fetchEvidence()
    } else {
      alert('上傳失敗。')
    }

  } catch (err) {
    alert('上傳發生錯誤：\n' + err.message)
    console.error(err)
  } finally {
    // 恢復初始狀態
    isUploading.value = false
    uploadProgress.value = 0
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

    <!-- 隱藏的檔案選擇器 -->
    <input 
      ref="fileInput" 
      type="file" 
      multiple 
      accept="audio/*,video/mp4" 
      class="hidden" 
      @change="handleBatchUpload" 
    />

    <!-- 動態狀態按鈕 -->
    <button 
      @click="$refs.fileInput.click()"
      :disabled="isUploading !== false"
      class="w-full py-3 mb-4 border-2 border-dashed border-blue-300 rounded-xl text-blue-600 font-medium hover:bg-blue-50 active:bg-blue-100 transition-colors"
    >
      <span v-if="isUploading === false">＋ 點擊選取手機錄音補傳 (可批次多選)</span>
      <span v-else-if="isUploading === true">正在傳送至伺服器 ({{ uploadProgress }}%)...</span>
      <span v-else-if="isUploading === 'processing'">☁️ 檔案已抵達，後台正轉存至 Telegram...</span>
    </button>

    <!-- 證據列表 (三按鈕排版) -->
    <ul class="space-y-3">
      <li v-for="item in evidenceList" :key="item.id" class="flex flex-col bg-gray-50 p-3 rounded-lg border border-gray-200 gap-3">
        <span class="text-sm font-bold text-gray-800 break-all">{{ item.title }}</span>
        
        <div class="flex flex-wrap gap-2">
          
          <!-- 立即串流下載 (安全產生網址，避免無 message_id 時產生錯誤) -->
          <a 
            v-if="item.message_id"
            :href="getDownloadUrl(item.message_id)" 
            target="_blank" 
            class="flex-1 text-center px-2 py-2 bg-green-500 text-white rounded-lg text-xs font-bold hover:bg-green-600 shadow-sm whitespace-nowrap"
          >
            ▶️ 立即串流下載🔗
          </a>

          <!-- TG 原文連結 -->
          <a 
            :href="item.telegram_url" 
            target="_blank" 
            class="flex-1 text-center px-2 py-2 bg-blue-500 text-white rounded-lg text-xs font-bold hover:bg-blue-600 shadow-sm whitespace-nowrap"
          >
            ✈️ TG 原文
          </a>

          <!-- 刪除按鈕 -->
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

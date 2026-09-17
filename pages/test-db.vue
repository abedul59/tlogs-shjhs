<script setup>
import { ref } from 'vue'

const supabase = useSupabaseClient()
const isTesting = ref(false)
const testResult = ref(null)
const errorMsg = ref(null)
const responseTime = ref(0)

const testDatabase = async () => {
  isTesting.value = true
  testResult.value = null
  errorMsg.value = null
  responseTime.value = 0

  const startTime = performance.now()

  try {
    // 測試讀取已知存在的資料表 'evidence_logs'，只抓 1 筆減輕負擔
    const { data, error } = await supabase
      .from('evidence_logs')
      .select('*')
      .limit(1)

    const endTime = performance.now()
    responseTime.value = Math.round(endTime - startTime)

    if (error) throw error

    testResult.value = data
  } catch (err) {
    console.error('Supabase 連線測試失敗:', err)
    errorMsg.value = err.message || JSON.stringify(err)
  } finally {
    isTesting.value = false
  }
}
</script>

<template>
  <div class="max-w-2xl mx-auto mt-10 p-6 bg-white rounded-xl shadow-md border border-gray-200">
    <h1 class="text-2xl font-bold text-gray-800 mb-2">🔌 Supabase 狀態檢測</h1>
    <p class="text-gray-600 mb-6 text-sm">
      測試剛喚醒的 Supabase 資料庫連線狀態。此測試將嘗試讀取 <code>evidence_logs</code> 資料表。
    </p>

    <button
      @click="testDatabase"
      :disabled="isTesting"
      class="w-full py-3 bg-blue-600 text-white font-bold rounded-lg hover:bg-blue-700 disabled:opacity-50 transition-colors flex justify-center items-center gap-2"
    >
      <span v-if="isTesting">🔄 正在連線測試中，請稍候...</span>
      <span v-else>🚀 開始測試資料庫連線</span>
    </button>

    <!-- 測試結果顯示區 -->
    <div v-if="testResult !== null || errorMsg" class="mt-6 p-5 rounded-lg transition-all" 
         :class="errorMsg ? 'bg-red-50 border border-red-200' : 'bg-green-50 border border-green-200'">
      
      <!-- 失敗狀態 -->
      <div v-if="errorMsg">
        <h3 class="text-lg font-bold text-red-700 mb-2 flex items-center gap-2">
          <span>❌</span> 連線失敗
        </h3>
        <p class="text-sm text-red-600 mb-3">
          資料庫可能還在喚醒中，或者 API 連線有問題。請參考以下錯誤訊息：
        </p>
        <pre class="bg-red-100 p-3 rounded-md text-sm text-red-800 overflow-x-auto whitespace-pre-wrap">{{ errorMsg }}</pre>
      </div>

      <!-- 成功狀態 -->
      <div v-else>
        <h3 class="text-lg font-bold text-green-700 mb-2 flex items-center gap-2">
          <span>✅</span> 資料庫連線正常！
        </h3>
        <ul class="text-sm text-green-800 space-y-1 mb-3">
          <li>• <strong>API 回應時間：</strong> {{ responseTime }} 毫秒</li>
          <li>• <strong>測試目標：</strong> 成功存取 <code>evidence_logs</code></li>
        </ul>
        
        <p class="text-xs text-green-700 mb-1 font-bold">回傳資料預覽 (JSON)：</p>
        <pre class="bg-green-100 p-3 rounded-md text-xs text-green-900 overflow-x-auto max-h-60 overflow-y-auto">
{{ testResult.length === 0 ? '[] (資料表為空，但連線成功)' : JSON.stringify(testResult, null, 2) }}
        </pre>
      </div>
      
    </div>
  </div>
</template>

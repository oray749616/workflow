<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted } from 'vue'
import { marked } from 'marked'
import { message } from 'ant-design-vue'

// 配置marked选项，禁用可能导致数字解析问题的功能
marked.setOptions({
  gfm: false, // 禁用GitHub风格Markdown
  pedantic: false
})

interface FormDataObject {
  model_id: string;
  text: string;
  channels: string;
  direction: string;
  requirements: string;
  num: string;
  seo_keywords: string;
  scope: string;
}

// 定义事件
const emit = defineEmits(['generation-complete'])

// 定义props接收响应数据和流式URL
const props = defineProps<{
  responseData?: {
    success: boolean
    data: string
  },
  streamUrl?: string,
  isStreaming?: boolean,
  formData?: FormDataObject
}>()

// 状态变量
const formattedContent = ref('')
const rawContent = ref('')
const loading = ref(false)
const error = ref('')
let eventSource: EventSource | null = null

// 清除事件源的函数
const cleanupEventSource = () => {
  if (eventSource) {
    eventSource.close()
    eventSource = null
  }
}

// 自定义创建EventSource的函数，使用POST方法
const createEventSourceWithPost = async (url: string, formData: FormDataObject) => {
  try {
    // 转换为FormData
    const formDataObj = new FormData()
    for (const [key, value] of Object.entries(formData)) {
      formDataObj.append(key, value)
    }
    
    // 发送POST请求
    const response = await fetch(url, {
      method: 'POST',
      body: formDataObj
    })
    
    if (!response.ok) {
      throw new Error('创建流式连接失败')
    }
    
    // 从响应头或响应体中读取连接信息
    const reader = response.body?.getReader()
    if (!reader) {
      throw new Error('无法读取响应流')
    }
    
    // 处理流式响应
    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      
      // 将二进制数据转换为文本
      const text = new TextDecoder().decode(value)
      
      // 处理SSE格式的数据
      const lines = text.split('\n')
      for (const line of lines) {
        if (line.startsWith('data: ')) {
          try {
            const eventData = JSON.parse(line.substring(6))
            
            // 根据事件类型处理
            switch (eventData.type) {
              case 'start':
                // 开始接收数据
                break
              case 'chunk':
                // 添加内容块
                rawContent.value += eventData.content
                // 预处理年份数字
                const processedContent = preprocessContent(rawContent.value)
                // 实时解析Markdown
                formattedContent.value = await marked.parse(processedContent)
                break
              case 'end':
                // 完成接收
                loading.value = false
                // 最终处理一次内容，确保年份格式正确
                const finalProcessedContent = preprocessContent(rawContent.value)
                formattedContent.value = await marked.parse(finalProcessedContent)
                // 通知生成完成
                emit('generation-complete')
                reader.cancel()
                return
              case 'error':
                // 处理错误
                error.value = eventData.message || '发生错误'
                loading.value = false
                // 通知生成完成(错误也是完成)
                emit('generation-complete')
                reader.cancel()
                return
            }
          } catch (e) {
            console.error('解析事件数据错误', e)
          }
        }
      }
    }
  } catch (e) {
    console.error('流式连接错误', e)
    error.value = e instanceof Error ? e.message : '连接错误'
    loading.value = false
    // 通知生成完成(错误也是完成)
    emit('generation-complete')
  }
}

// 预处理内容，保护年份格式
const preprocessContent = (content: string): string => {
  // 对可能有问题的年份数字添加保护
  // 匹配四位数字后跟"年"字的模式，用HTML实体替换数字
  return content.replace(/(\d{4})年/g, (match, year) => {
    // console.log(`捕获年份: ${year}年`)
    return `${year}年`
  })
}

// 处理常规响应数据
watch(() => props.responseData, async (newVal) => {
  if (newVal?.success) {
    rawContent.value = newVal.data
    // 预处理并解析
    const processedContent = preprocessContent(rawContent.value)
    formattedContent.value = await marked.parse(processedContent)
  }
}, { immediate: true })

// 处理流式数据
watch(() => [props.streamUrl, props.isStreaming, props.formData], async (newValues) => {
  // 清除旧的事件源
  cleanupEventSource()
  
  const newUrl = newValues[0] as string | undefined
  const isStreaming = newValues[1] as boolean | undefined
  const formData = newValues[2] as FormDataObject | undefined
  
  if (newUrl && isStreaming && formData) {
    // 重置状态
    rawContent.value = ''
    formattedContent.value = ''
    error.value = ''
    loading.value = true
    
    // 使用POST方式创建流式连接
    await createEventSourceWithPost(newUrl, formData)
  }
}, { immediate: true })

// 复制原始Markdown内容到剪贴板
const copyContent = async () => {
  try {
    await navigator.clipboard.writeText(rawContent.value)
    message.success('内容已复制到剪贴板')
  } catch (e) {
    message.error('复制失败，请重试')
  }
}

// 复制纯文本内容到剪贴板（移除Markdown格式）
const copyPlainText = async () => {
  try {
    // 创建临时DOM元素来提取纯文本内容
    const tempDiv = document.createElement('div')
    tempDiv.innerHTML = formattedContent.value
    const plainText = tempDiv.textContent || tempDiv.innerText || ''
    
    await navigator.clipboard.writeText(plainText)
    message.success('纯文本内容已复制到剪贴板')
  } catch (e) {
    message.error('复制失败，请重试')
  }
}

// 组件卸载时清理
onUnmounted(() => {
  cleanupEventSource()
})
</script>

<template>
  <div class="response-container bg-gray-50 p-4 rounded-lg h-full overflow-auto">

    <div v-if="loading" class="text-center py-4">
      <div class="inline-block animate-pulse bg-blue-100 text-blue-800 px-4 py-2 rounded-full">
        正在生成内容...
      </div>
    </div>

    <div v-if="error" class="text-red-500 mb-4">
      {{ error }}
    </div>



    <div class="markdown-content" v-html="formattedContent"></div>

    <!-- 按钮容器 -->
    <div v-if="!loading && rawContent" class="fixed bottom-6 right-6 flex flex-col gap-2">
      <!-- 复制Markdown格式内容 -->
      <button @click="copyContent"
        class="px-4 py-2 bg-blue-500 hover:bg-blue-600 text-white rounded-full shadow-lg text-sm transition cursor-pointer flex items-center"
        title="复制Markdown格式内容">
        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 5H6a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2v-1M8 5a2 2 0 002 2h2a2 2 0 002-2M8 5a2 2 0 012-2h2a2 2 0 012 2m0 0h2a2 2 0 012 2v3m2 4H10m0 0l3-3m-3 3l3 3" />
        </svg>
        复制Markdown
      </button>

      <!-- 复制纯文本内容 -->
      <button @click="copyPlainText"
        class="px-4 py-2 bg-green-500 hover:bg-green-600 text-white rounded-full shadow-lg text-sm transition cursor-pointer flex items-center"
        title="复制纯文本内容（无格式）">
        <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 mr-1" fill="none" viewBox="0 0 24 24" stroke="currentColor">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z" />
        </svg>
        复制纯文本
      </button>
    </div>
  </div>
</template>

<style scoped>
.response-container {
  max-height: 780px;
  min-height: 780px;
  border: 1px solid #e5e7eb;
  overflow-y: auto;
  position: relative;
}

.markdown-content {
  text-align: left;
  white-space: pre-wrap;
  padding-bottom: 140px; /* 为三个悬浮按钮留出空间 */
  
  /* Markdown内容样式 */
  h1, h2, h3, h4, h5, h6 {
    margin: 1em 0;
    font-weight: bold;
  }
  
  p {
    margin: 1em 0;
    line-height: 1.6;
  }
  
  ul, ol {
    padding-left: 2em;
    margin: 1em 0;
  }
  
  code {
    background-color: #f3f4f6;
    padding: 0.2em 0.4em;
    border-radius: 0.25em;
    font-family: monospace;
  }
  
  pre {
    background-color: #f3f4f6;
    padding: 1em;
    border-radius: 0.5em;
    overflow: auto;
  }
  
  blockquote {
    border-left: 4px solid #e5e7eb;
    padding-left: 1em;
    margin: 1em 0;
    color: #6b7280;
  }
}
</style>
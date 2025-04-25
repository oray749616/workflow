<script setup lang="ts">
import { ref } from 'vue'
import type { NewsFormDataObject } from '@/types/newsFormDataObject'

definePageMeta({
  layout: 'home',
})

// 表单组件引用
const formComponentRef = ref()
// 响应数据
const responseData = ref({ success: false, data: '' })
// 流式传输状态
const streamState = ref({
  streamUrl: '',
  isStreaming: false,
  formData: undefined as NewsFormDataObject | undefined
})

// 处理响应数据更新
const handleResponse = (data: { success: boolean; data: string } | { success: boolean; data: string }) => {
  responseData.value = data
  // 收到普通响应时，确保流式状态关闭
  streamState.value.isStreaming = false
}

// 处理流式传输开始
const handleStreamStart = (data: { streamUrl: string; isStreaming: boolean; formData: NewsFormDataObject }) => {
  streamState.value = {
    streamUrl: data.streamUrl,
    isStreaming: data.isStreaming,
    formData: data.formData
  }
  // 清空之前的响应数据
  responseData.value = { success: false, data: '' }
}

// 处理生成完成事件
const handleGenerationComplete = () => {
  // 通知表单组件生成已完成
  if (formComponentRef.value && formComponentRef.value.onGenerationComplete) {
    formComponentRef.value.onGenerationComplete()
  }
}
</script>

<template>
  <h1 class="text-3xl text-bold mb-6">文案生成工具</h1>
  <div class="flex gap-4 p-4 mx-auto" style="max-width: 1600px;">
    <FormComponent 
      ref="formComponentRef"
      @response-updated="handleResponse" 
      @stream-started="handleStreamStart"
      class="flex-1" 
    />
    <DisplayComponent 
      :response-data="responseData" 
      :stream-url="streamState.streamUrl"
      :is-streaming="streamState.isStreaming"
      :form-data="streamState.formData"
      @generation-complete="handleGenerationComplete"
      class="flex-1" 
    />
  </div>
</template>
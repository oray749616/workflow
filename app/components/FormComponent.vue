<script setup lang="ts">
import { ref } from 'vue'
const emit = defineEmits(['response-updated', 'stream-started'])

// 表单引用
const formRef = ref<any>(null)

// 表单数据模型
const formState = ref({
  model_id: 'deepseek-v3-250324',
  text: '',
  channels: [],
  direction: '',
  requirements: '',
  num: 1,
  seo_keywords: '',
  scope: '',
  useStreaming: true // 默认启用流式传输
})

// 提交状态
const submitting = ref(false)

// 表单提交处理函数
const handleSubmit = async () => {
  if (submitting.value) return

  try {
    // 先进行表单验证
    if (formRef.value) {
      await formRef.value.validate()
    } else {
      message.error('表单初始化错误')
      return
    }
    
    submitting.value = true
    
    // 准备数据对象
    const formData = {
      model_id: formState.value.model_id,
      text: formState.value.text,
      channels: formState.value.channels.join('、'),
      direction: formState.value.direction,
      requirements: formState.value.requirements,
      num: formState.value.num.toString(),
      seo_keywords: formState.value.seo_keywords,
      scope: formState.value.scope
    }

    if (formState.value.useStreaming) {
      // 流式API
      message.success('开始流式生成内容')
      
      // 通知父组件开始流式传输
      emit('stream-started', {
        streamUrl: '/api/deepseek/streamChat',
        isStreaming: true,
        formData: formData
      })
    } else {
      // 常规API - 转换为FormData以兼容现有代码
      const formDataObj = new FormData()
      for (const [key, value] of Object.entries(formData)) {
        formDataObj.append(key, value)
      }
      
      const response = await $fetch('/api/deepseek/chat', {
        method: 'POST',
        body: formDataObj
      })
      
      // 处理响应
      console.log('提交成功:', response)
      message.success('提交成功')
      // 触发响应数据更新
      emit('response-updated', response)
      submitting.value = false
    }
  } catch (error: any) {
    console.error('操作失败:', error)
    
    // 检查是否为表单验证错误
    if (error && typeof error === 'object' && 'errorFields' in error) {
      message.error('请填写完整的表单信息')
    } else {
      message.error('提交失败')
    }
    
    submitting.value = false
  }
}

// 接收生成完成的事件
const onGenerationComplete = () => {
  submitting.value = false
}

// 向父组件暴露方法
defineExpose({
  onGenerationComplete
})
</script>

<template>
  <div>
    <a-form ref="formRef" :model="formState" layout="vertical" class="max-w-2xl mx-auto">
      <!-- Model ID -->
      <a-form-item label="选择模型" name="model_id" :rules="[{ required: true, message: '请选择模型' }]" class="mb-4">
        <a-select v-model:value="formState.model_id" placeholder="请选择模型">
          <a-select-option value="deepseek-v3-250324">deepseek-v3</a-select-option>
          <a-select-option value="deepseek-r1-distill-qwen-32b-250120">deepseek-r1</a-select-option>
          <a-select-option value="doubao-1-5-thinking-pro-250415">doubao-1-5-thinking-pro</a-select-option>
          <a-select-option value="doubao-1.5-vision-pro-250328">doubao-1-5-vision-pro</a-select-option>
        </a-select>
      </a-form-item>

      <!-- 文本内容 -->
      <a-form-item label="文本内容" name="text" :rules="[{ required: true, message: '请输入文本内容' }]" class="mb-4">
        <a-textarea v-model:value="formState.text" :auto-size="{ minRows: 4, maxRows: 10 }"
          placeholder="在迪拜哈利法塔的云端餐厅，伦敦碎片大厦的奢华大堂，东京六本木之丘的艺术空间，一种源自中国佛山的瓷砖正以卓越品质征服全球高端建筑市场。伊莉莎白瓷砖（Elizabeth Tiles），这个诞生于世界陶瓷之都的品牌，凭借其革命性的生产工艺和智能制造体系，正在重新定义国际建筑装饰标准。" />
      </a-form-item>

      <!-- 推广渠道 -->
      <a-form-item label="推广渠道" name="channels" :rules="[{ required: true, message: '请选择推广渠道' }]" class="mb-4">
        <a-select mode="tags" v-model:value="formState.channels" placeholder="请选择渠道">
          <a-select-option value="百家号">百家号</a-select-option>
          <a-select-option value="头条号">头条号</a-select-option>
          <a-select-option value="抖音">抖音</a-select-option>
          <a-select-option value="小红书">小红书</a-select-option>
        </a-select>
      </a-form-item>

      <!-- 内容方向 -->
      <a-form-item label="内容方向" name="direction" :rules="[{ required: true, message: '请输入内容方向' }]" class="mb-4">
        <a-input v-model:value="formState.direction" placeholder="如：关于国际品牌的推广文章" />
      </a-form-item>

      <!-- 说明要求 -->
      <a-form-item label="说明要求" name="requirements" :rules="[{ required: true, message: '请输入说明要求' }]" class="mb-4">
        <a-input v-model:value="formState.requirements" placeholder="如：内容大意不能变化" />
      </a-form-item>

      <!-- SEO关键词 -->
      <a-form-item label="SEO关键词(以顿号隔开)" name="seo_keywords" :rules="[{ required: true, message: '请输入SEO关键词' }]"
        class="mb-4">
        <a-input v-model:value="formState.seo_keywords" placeholder="如：国际瓷砖品牌" />
      </a-form-item>

      <!-- 内容作用 -->
      <a-form-item label="内容作用" name="scope" :rules="[{ required: true, message: '请输入内容作用' }]" class="mb-4">
        <a-input v-model:value="formState.scope" placeholder="如：百度搜索引擎收录、品牌推广、媒体推广" />
      </a-form-item>

      <a-form-item>
        <a-button type="primary" @click="handleSubmit" class="w-full bg-blue-500 hover:bg-blue-600"
          :loading="submitting" :disabled="submitting">
          {{ submitting ? '生成中...' : '提交' }}
        </a-button>
      </a-form-item>
    </a-form>
  </div>
</template>
<template>
    <a-card title="gRPC 任务配置" :bordered="false">
      <a-form
        ref="formRef"
        :model="form"
        :rules="rules"
        layout="vertical"
        @submit.prevent="handleSubmit"
      >
        
        <a-divider orientation="left">基础信息</a-divider>
        <a-form-item label="Host" name="host">
          <a-input v-model:value="form.host" placeholder="127.0.0.1" />
        </a-form-item>
        <a-form-item label="Port" name="port">
          <a-input-number v-model:value="form.port" :min="1" :max="65535" style="width: 100%" />
        </a-form-item>
        <a-form-item label="Method" name="method">
          <a-input v-model:value="form.method" placeholder="YourService.YourMethod" />
        </a-form-item>
  
      
        <a-divider orientation="left">请求参数 (JSON)</a-divider>
        <a-form-item name="requestJson">
          <a-textarea
            v-model:value="form.requestJson"
            rows="5"
            placeholder='{"name": "value"}'
          />
        </a-form-item>
  
    
        <a-divider orientation="left">gRPC Metadata</a-divider>
        <div v-for="(meta, index) in form.metadata" :key="index" class="meta-row">
          <a-input
            v-model:value="meta.key"
            placeholder="Key"
            style="width: 40%; margin-right: 8px"
          />
          <a-input
            v-model:value="meta.value"
            placeholder="Value"
            style="width: 40%; margin-right: 8px"
          />
          <a-button danger @click="removeMetadata(index)">删除</a-button>
        </div>
        <a-button type="dashed" @click="addMetadata" style="margin-top: 8px">
          添加 Header
        </a-button>
  
     
        <a-divider orientation="left">重试配置</a-divider>
        <a-form-item label="重试次数">
          <a-input-number v-model:value="form.retryCount" :min="0" style="width: 100%" />
        </a-form-item>
        <a-form-item label="重试间隔 (ms)">
          <a-input-number v-model:value="form.retryInterval" :min="0" style="width: 100%" />
        </a-form-item>
        <a-form-item label="重试策略">
          <a-select v-model:value="form.retryStrategy">
            <a-select-option value="fixed">固定</a-select-option>
            <a-select-option value="exponential">指数退避</a-select-option>
          </a-select>
        </a-form-item>


        <a-divider orientation="left">SSL/TLS</a-divider>
        <a-form-item>
          <a-switch v-model:checked="form.enableSSL" /> 启用 SSL
        </a-form-item>
        <div v-if="form.enableSSL">
          <a-form-item label="证书路径">
            <a-input v-model:value="form.certPath" placeholder="/path/to/cert.pem" />
          </a-form-item>
          <a-form-item>
            <a-checkbox v-model:checked="form.skipVerify">跳过验证</a-checkbox>
          </a-form-item>
        </div>
  
        <a-form-item>
          <a-button type="primary" html-type="submit"> 生成配置</a-button>
        </a-form-item>
  
        <a-divider orientation="left">生成的配置 JSON</a-divider>
        <a-alert
          v-if="generatedConfig"
          :message="'配置结果'"
          type="info"
          show-icon
          :description="generatedConfig"
        />
      </a-form>
    </a-card>
  </template>
  
  <script setup lang="ts">
  import { ref } from 'vue'
  import type { Rule } from 'ant-design-vue/es/form'
  
  interface Metadata {
    key: string
    value: string
  }
  
  interface GrpcForm {
    host: string
    port: number | null
    method: string
    requestJson: string
    metadata: Metadata[]
    retryCount: number
    retryInterval: number
    retryStrategy: 'fixed' | 'exponential'
    enableSSL: boolean
    certPath: string
    skipVerify: boolean
  }
  
  const formRef = ref()
  const form = ref<GrpcForm>({
    host: '',
    port: 50051,
    method: '',
    requestJson: '',
    metadata: [],
    retryCount: 3,
    retryInterval: 1000,
    retryStrategy: 'fixed',
    enableSSL: false,
    certPath: '',
    skipVerify: false,
  })
  
  // 表单校验规则
  const rules: Record<string, Rule[]> = {
    host: [{ required: true, message: '请输入 gRPC 服务地址' }],
    port: [{ required: true, message: '请输入端口号' }],
    method: [{ required: true, message: '请输入调用方法，例如 YourService.YourMethod' }],
    requestJson: [{ required: true, message: '请输入请求参数 JSON' }],
  }
  
  function addMetadata() {
    form.value.metadata.push({ key: '', value: '' })
  }
  function removeMetadata(index: number) {
    form.value.metadata.splice(index, 1)
  }
  
  const generatedConfig = ref('')
  function handleSubmit() {
    formRef.value?.validate().then(() => {
      try {
        const request = JSON.parse(form.value.requestJson || '{}')
        const config = {
          type: 'GRPC',
          host: form.value.host,
          port: form.value.port,
          method: form.value.method,
          request,
          metadata: Object.fromEntries(
            form.value.metadata.filter(m => m.key).map(m => [m.key, m.value])
          ),
          retry: {
            count: form.value.retryCount,
            interval: form.value.retryInterval,
            strategy: form.value.retryStrategy,
          },
          ssl: form.value.enableSSL
            ? {
                enabled: true,
                certPath: form.value.certPath,
                skipVerify: form.value.skipVerify,
              }
            : { enabled: false },
        }
        generatedConfig.value = JSON.stringify(config, null, 2)
      } catch (err) {
        generatedConfig.value = '请求参数 JSON 格式错误'
      }
    })
  }
  </script>
  
  <style scoped>
  .meta-row {
    display: flex;
    align-items: center;
    margin-bottom: 8px;
  }
  </style>
  
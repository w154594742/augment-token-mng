<template>
  <div class="proxy-config-modal">
    <div class="modal-overlay" @click="$emit('close')">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h2>代理设置</h2>
          <button class="close-btn" @click="$emit('close')">×</button>
        </div>
        
        <div class="modal-body">
          <div class="config-form">
            <!-- 启用代理开关 -->
            <div class="form-group">
              <label class="checkbox-label">
                <input type="checkbox" v-model="config.enabled" :disabled="isLoading">
                <span class="checkbox-text">启用代理</span>
              </label>
            </div>
            
            <!-- 代理配置选项 -->
            <template v-if="config.enabled">
              <div class="form-group">
                <label for="proxy-type">代理类型:</label>
                <select
                  id="proxy-type"
                  v-model="config.proxy_type"
                  :disabled="isLoading"
                >
                  <option value="http">HTTP</option>
                  <option value="socks5">SOCKS5</option>
                </select>
              </div>
              
              <div class="form-group">
                <label for="proxy-host">服务器地址:</label>
                <input
                  id="proxy-host"
                  v-model="config.host"
                  type="text"
                  placeholder="127.0.0.1"
                  :disabled="isLoading"
                >
              </div>
              
              <div class="form-group">
                <label for="proxy-port">端口:</label>
                <input
                  id="proxy-port"
                  v-model.number="config.port"
                  type="number"
                  placeholder="8080"
                  min="1"
                  max="65535"
                  :disabled="isLoading"
                >
              </div>
              
              <div class="form-group">
                <label for="proxy-username">用户名 (可选):</label>
                <input
                  id="proxy-username"
                  v-model="config.username"
                  type="text"
                  placeholder="用户名"
                  :disabled="isLoading"
                >
              </div>
              
              <div class="form-group">
                <label for="proxy-password">密码 (可选):</label>
                <input
                  id="proxy-password"
                  v-model="config.password"
                  type="password"
                  placeholder="密码"
                  :disabled="isLoading"
                >
              </div>
            </template>
          </div>
        </div>
        
        <div class="modal-footer">
          <button
            @click="testConnection"
            :class="['btn', 'secondary', { loading: isTesting }]"
            :disabled="!canTest || isTesting"
            v-if="config.enabled"
          >
            <svg v-if="!isTesting" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-2 15l-5-5 1.41-1.41L10 14.17l7.59-7.59L19 8l-9 9z"/>
            </svg>
            测试连接
          </button>
          
          <button
            @click="saveConfig"
            :class="['btn', 'primary', { loading: isSaving }]"
            :disabled="isSaving"
          >
            <svg v-if="!isSaving" width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <path d="M17 3H5c-1.11 0-2 .9-2 2v14c0 1.1.89 2 2 2h14c1.1 0 2-.9 2-2V7l-4-4zm-5 16c-1.66 0-3-1.34-3-3s1.34-3 3-3 3 1.34 3 3-1.34 3-3 3zm3-10H5V5h10v4z"/>
            </svg>
            保存配置
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import { invoke } from '@tauri-apps/api/core'

// 组件事件定义
const emit = defineEmits(['close', 'show-status', 'config-saved'])

// 响应式数据
const config = ref({
  enabled: false,
  proxy_type: 'http',
  host: '127.0.0.1',
  port: 8080,
  username: '',
  password: ''
})

const isLoading = ref(false)
const isTesting = ref(false)
const isSaving = ref(false)
const isConnectionTested = ref(false)

// 计算属性
const canTest = computed(() => {
  return config.value.enabled && 
         config.value.host && 
         config.value.port > 0 && 
         config.value.port <= 65535
})

// 方法定义
const loadConfig = async () => {
  isLoading.value = true
  try {
    console.log('Calling load_proxy_config...')
    const loadedConfig = await invoke('load_proxy_config')
    console.log('load_proxy_config result:', loadedConfig)
    if (loadedConfig) {
      config.value = {
        enabled: loadedConfig.enabled || false,
        proxy_type: loadedConfig.proxy_type || 'http',
        host: loadedConfig.host || '127.0.0.1',
        port: loadedConfig.port || 8080,
        username: loadedConfig.username || '',
        password: loadedConfig.password || ''
      }
    }
  } catch (error) {
    console.error('Failed to load proxy config:', error)
    // 不显示错误消息，使用默认配置
    config.value = {
      enabled: false,
      proxy_type: 'http',
      host: '127.0.0.1',
      port: 8080,
      username: '',
      password: ''
    }
  } finally {
    isLoading.value = false
  }
}


const testConnection = async () => {
  isTesting.value = true
  try {
    await invoke('test_proxy_connection', {
      proxyType: config.value.proxy_type,
      host: config.value.host,
      port: config.value.port,
      username: config.value.username || null,
      password: config.value.password || null
    })
    
    isConnectionTested.value = true
    emit('show-status', '代理连接测试成功！', 'success')
  } catch (error) {
    isConnectionTested.value = false
    emit('show-status', `代理连接测试失败: ${error}`, 'error')
  } finally {
    isTesting.value = false
  }
}

const saveConfig = async () => {
  isSaving.value = true
  try {
    console.log('Calling save_proxy_config with:', {
      enabled: config.value.enabled,
      proxyType: config.value.proxy_type,
      host: config.value.host,
      port: config.value.port,
      username: config.value.username || null,
      password: config.value.password || null
    })

    await invoke('save_proxy_config', {
      enabled: config.value.enabled,
      proxyType: config.value.proxy_type,
      host: config.value.host,
      port: config.value.port,
      username: config.value.username || null,
      password: config.value.password || null
    })
    
    emit('show-status', '代理配置保存成功！', 'success')
    emit('config-saved')
    emit('close')
  } catch (error) {
    console.error('Save proxy config error:', error)
    emit('show-status', `保存代理配置失败: ${error}`, 'error')
  } finally {
    isSaving.value = false
  }
}

// 监听配置变化，重置连接测试状态
watch(config, () => {
  isConnectionTested.value = false
}, { deep: true })

// 组件挂载时加载配置
onMounted(async () => {
  try {
    await loadConfig()
  } catch (error) {
    console.error('Failed to load config on mount:', error)
    // 如果加载失败，使用默认配置
    config.value = {
      enabled: false,
      proxy_type: 'http',
      host: '127.0.0.1',
      port: 8080,
      username: '',
      password: ''
    }
  }
})
</script>

<style scoped>
/* 复用数据库配置的样式 */
.proxy-config-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}

.modal-content {
  background: white;
  border-radius: 12px;
  max-width: 500px;
  width: 90vw;
  max-height: 90vh;
  overflow-y: auto;
  position: relative;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 24px;
  border-bottom: 1px solid #e5e7eb;
  background: #f9fafb;
  border-radius: 12px 12px 0 0;
}

.modal-header h2 {
  margin: 0;
  color: #374151;
  font-size: 18px;
  font-weight: 600;
}

.close-btn {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #6b7280;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.2s;
}

.close-btn:hover {
  background: #e5e7eb;
  color: #374151;
}

.modal-body {
  padding: 24px;
}

.config-form {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.form-group label {
  font-weight: 500;
  color: #374151;
  font-size: 14px;
}

.checkbox-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  flex-direction: row !important;
}

.checkbox-text {
  font-weight: 500;
  color: #374151;
  font-size: 14px;
}

.form-group input,
.form-group select {
  padding: 10px 12px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 14px;
  transition: border-color 0.2s ease;
}

.form-group input:focus,
.form-group select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.form-group input:disabled,
.form-group select:disabled {
  background-color: #f9fafb;
  color: #6b7280;
  cursor: not-allowed;
}

.modal-footer {
  display: flex;
  gap: 12px;
  padding: 20px 24px;
  border-top: 1px solid #e5e7eb;
  background: #f9fafb;
  border-radius: 0 0 12px 12px;
  justify-content: flex-end;
}

.btn {
  padding: 10px 16px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.btn.primary {
  background: #3b82f6;
  color: white;
}

.btn.primary:hover {
  background: #2563eb;
}

.btn.secondary {
  background: #6b7280;
  color: white;
}

.btn.secondary:hover {
  background: #4b5563;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
  pointer-events: none;
}

.btn.loading {
  opacity: 0.7;
  cursor: wait;
}
</style>

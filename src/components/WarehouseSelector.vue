<template>
  <div class="warehouse-selector">
    <el-select v-model="selectedWarehouse" placeholder="主仓库" @change="handleWarehouseChange">
      <el-option
        v-for="item in warehouseList"
        :key="item.id"
        :label="item.name"
        :value="item.id"
      />
    </el-select>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { ElSelect, ElOption } from 'element-plus'

const selectedWarehouse = ref('1')

// 仓库列表
const warehouseList = [
  { id: '1', name: '主仓库' },
  { id: '2', name: '备用仓库' }
]

const emit = defineEmits(['change'])

const handleWarehouseChange = (value) => {
  emit('change', value)
}
</script>

<style scoped>
.warehouse-selector {
  display: flex;
  justify-content: center;
  padding: 12px 24px;
  position: relative;
}

.el-select {
  width: 240px;
  --el-select-bg-color: rgba(26, 41, 64, 0.8);
  --el-select-border-color-hover: #409EFF;
  --el-select-input-focus-border-color: #409EFF;
  --el-select-text-color: #ffffff;
  backdrop-filter: blur(20px);
  border-radius: 12px;
  box-shadow: 
    0 4px 24px rgba(0, 0, 0, 0.2),
    0 0 2px rgba(64, 158, 255, 0.3),
    inset 0 0 20px rgba(64, 158, 255, 0.05);
  border: 1px solid rgba(64, 158, 255, 0.15);
  transition: all 0.4s cubic-bezier(0.4, 0, 0.2, 1);
}

.el-select:hover {
  border-color: rgba(64, 158, 255, 0.3);
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.3),
    0 0 4px rgba(64, 158, 255, 0.4),
    inset 0 0 32px rgba(64, 158, 255, 0.08);
  transform: translateY(-2px);
}

:deep(.el-select-dropdown) {
  background-color: rgba(26, 41, 64, 0.95);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(64, 158, 255, 0.15);
  border-radius: 12px;
  box-shadow: 
    0 8px 32px rgba(0, 0, 0, 0.3),
    0 0 4px rgba(64, 158, 255, 0.3);
  padding: 8px;
}

:deep(.el-select-dropdown__item) {
  color: #ffffff;
  transition: all 0.3s ease;
  border-radius: 8px;
  margin: 2px 0;
}

:deep(.el-select-dropdown__item.hover) {
  background-color: rgba(64, 158, 255, 0.1);
}

:deep(.el-select-dropdown__item.selected) {
  background-color: rgba(64, 158, 255, 0.2);
  color: #409EFF;
  text-shadow: 0 0 20px rgba(64, 158, 255, 0.3);
}

:deep(.el-input__wrapper) {
  background-color: transparent;
  box-shadow: none !important;
}

:deep(.el-input__inner) {
  color: #ffffff;
  text-shadow: 0 0 20px rgba(64, 158, 255, 0.2);
}
</style>
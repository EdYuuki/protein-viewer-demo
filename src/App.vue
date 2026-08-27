<template>
  <div v-if="loading" class="loading">蛋白结构加载中...</div>

  <div class="demo-wrap">
    <h2>蛋白结构可视化Demo</h2>
    
    <div class="control-bar">
      <div class="info-panel">
       <span>当前蛋白：{{ currentProtein }}</span>
       <span>渲染模式：{{ currentMode }}</span>
      </div>
      <input type="file" accept=".pdb,.cif" @change="loadLocalFile" style="margin-left:auto" />
      <button @click="showCartoon">卡通模式</button>
      <button @click="showSpacefill">球体模式</button>
      <button @click="resetView">重置视角</button>
      <button @click="exportImage">导出截图</button>
    </div>
    <div ref="stageRef" class="stage-box"></div>
    <p class="tip">默认加载测试蛋白 1AKE | 左键旋转、滚轮缩放、右键平移</p>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue'

const stageRef = ref(null)
let stage = null
let proteinComponent = null
const loading = ref(false)

const currentProtein = ref("1AKE (RCSB在线)")
const currentMode = ref("cartoon")

onMounted(async () => {
  const NGL = await import('ngl')
  await nextTick()
  if (!stageRef.value) return

  stage = new NGL.Stage(stageRef.value, { backgroundColor: '#f5f7fa' })

  loading.value = true
  stage.loadFile('rcsb://1AKE')
    .then(component => {
      proteinComponent = component
      component.addRepresentation('cartoon', { color: 'chainid' })
      component.autoView()
      loading.value = false
    })
    .catch(err => {
      console.error('蛋白加载失败:', err)
      alert('蛋白加载失败，请检查网络连接（需要访问RCSB数据库）')
      loading.value = false
    })
})

// 组件销毁，释放NGL资源，防止热更新内存泄露
onUnmounted(() => {
  if(stage){
    stage.dispose()
    stage = null
  }
})

const showCartoon = () => {
  if (!proteinComponent) return
  proteinComponent.removeAllRepresentations()
  proteinComponent.addRepresentation('cartoon', { color: 'chainid' })
  currentMode.value = "cartoon"
}

const showSpacefill = () => {
  if (!proteinComponent) return
  proteinComponent.removeAllRepresentations()
  proteinComponent.addRepresentation('spacefill', { opacity: 0.6 })
  currentMode.value = "spacefill"
}

const resetView = () => {
  proteinComponent?.autoView()
}

const exportImage = () => {
  if (!stage) return
  stage.makeImage({ antialias: true }).then(blob => {
    const link = document.createElement('a')
    link.href = URL.createObjectURL(blob)
    link.download = 'protein-structure.png'
    link.click()
  })
}

/**
 * ✔修复本地文件上传逻辑
 * NGL loadFile：传File对象，而不是readAsText文本；自动识别后缀pdb/cif
 */
const loadLocalFile = (e) => {
  const file = e.target.files[0]
  if (!file || !stage) return

  loading.value = true
  stage.removeAllComponents()

  // 直接传入 File 对象！不要 readAsText！
  stage.loadFile(file)
    .then(component => {
      proteinComponent = component
      component.addRepresentation('cartoon', { color: 'chainid' })
      component.autoView()
      currentProtein.value = file.name
      currentMode.value = "cartoon"
      loading.value = false
    })
    .catch(err => {
      console.error("本地文件解析失败", err)
      alert(`文件解析失败，请确认是合法pdb/cif：${err.message}`)
      loading.value = false
    })
}
</script>

<style scoped>
.demo-wrap {
  max-width: 1200px;
  margin: 20px auto;
  padding: 0 20px;
  font-family: system-ui, sans-serif;
}
.info-panel{
  display:flex;
  gap:24px;
  font-size:15px;
}
.control-bar {
  margin: 15px 0;
  display: flex;
  gap: 10px;
}
.control-bar button {
  padding: 6px 14px;
  cursor: pointer;
  border: 1px solid #dcdfe6;
  background: #fff;
  border-radius: 4px;
}
.control-bar button:hover {
  border-color: #409eff;
  color: #409eff;
}
.stage-box {
  width: 100%;
  height: 600px;
  border: 1px solid #e4e7ed;
  border-radius: 4px;
}
.tip {
  margin-top: 10px;
  color: #909399;
  font-size: 14px;
}
</style>

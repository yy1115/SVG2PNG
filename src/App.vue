<template>
  <div class="container">
    <el-row :gutter="20">
      <el-col :span="12">
        <el-card class="input-card">
          <template #header>
            <div class="card-header">
              <span>SVG代码</span>
            </div>
          </template>
          <el-input
            v-model="svgCode"
            type="textarea"
            :rows="10"
            placeholder="请输入SVG代码"
            @input="updatePreview"
          />
        </el-card>
      </el-col>
      <el-col :span="12">
        <el-card class="preview-card">
          <template #header>
            <div class="card-header">
              <span>预览</span>
              <el-button type="primary" @click="saveImage" :disabled="!svgCode">
                保存图片
              </el-button>
            </div>
          </template>
          <div class="preview-container" ref="previewContainer" v-html="svgCode"></div>
        </el-card>
      </el-col>
    </el-row>
    
    <el-row :gutter="20" style="margin-top: 20px">
      <el-col :span="24">
        <el-card>
          <template #header>
            <div class="card-header">
              <span>导出设置</span>
            </div>
          </template>
          <el-form :model="exportSettings" label-width="120px">
            <el-form-item label="文件名">
              <el-input v-model="exportSettings.filename" placeholder="输入文件名（不含扩展名）"></el-input>
            </el-form-item>
            <el-form-item label="图片格式">
              <el-radio-group v-model="exportSettings.format">
                <el-radio label="png">PNG</el-radio>
                <el-radio label="jpeg">JPEG</el-radio>
              </el-radio-group>
            </el-form-item>
            <el-form-item label="背景颜色" v-if="exportSettings.format === 'jpeg'">
              <el-color-picker v-model="exportSettings.backgroundColor"></el-color-picker>
            </el-form-item>
            <el-form-item label="缩放比例">
              <el-slider v-model="exportSettings.scale" :min="0.5" :max="3" :step="0.1" show-input></el-slider>
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script setup>
import { ref, onMounted, reactive } from 'vue'
import { saveAs } from 'file-saver'
import { ElMessage } from 'element-plus'

const svgCode = ref('')
const previewContainer = ref(null)
const svgError = ref('')

const exportSettings = reactive({
  filename: 'svg-image',
  format: 'png',
  backgroundColor: '#ffffff',
  scale: 1
})

const updatePreview = () => {
  // 清除之前的错误信息
  svgError.value = ''
  
  if (!svgCode.value.trim()) {
    return
  }
  
  try {
    // 创建一个临时的DOM解析器来验证SVG代码
    const parser = new DOMParser()
    const doc = parser.parseFromString(svgCode.value, 'image/svg+xml')
    
    // 检查解析错误
    const parserError = doc.querySelector('parsererror')
    if (parserError) {
      // 提取错误信息
      const errorText = parserError.textContent
      svgError.value = `SVG解析错误: ${errorText.split('\n')[0]}`
      return
    }
    
    // 检查根元素是否为SVG
    const rootElement = doc.documentElement
    if (rootElement.nodeName.toLowerCase() !== 'svg') {
      svgError.value = '无效的SVG: 根元素必须是<svg>标签'
      return
    }
    
    // 检查必要的属性
    if (!rootElement.hasAttribute('xmlns')) {
      svgError.value = '警告: 缺少xmlns属性，建议添加xmlns="http://www.w3.org/2000/svg"'
    }
  } catch (error) {
    svgError.value = `SVG验证错误: ${error.message}`
    console.error('SVG验证错误:', error)
  }
}

const saveImage = async () => {
  try {
    const svgElement = previewContainer.value.querySelector('svg')
    if (!svgElement) {
      ElMessage.error('无效的SVG代码')
      return
    }

    // 获取SVG的尺寸
    let width, height
    
    // 尝试从viewBox获取尺寸
    if (svgElement.getAttribute('viewBox')) {
      const viewBox = svgElement.getAttribute('viewBox').split(' ')
      width = parseFloat(viewBox[2])
      height = parseFloat(viewBox[3])
    } 
    // 尝试从width/height属性获取尺寸
    else if (svgElement.getAttribute('width') && svgElement.getAttribute('height')) {
      width = parseFloat(svgElement.getAttribute('width'))
      height = parseFloat(svgElement.getAttribute('height'))
    } 
    // 使用getBBox获取尺寸
    else {
      const bbox = svgElement.getBBox()
      width = bbox.width
      height = bbox.height
    }
    
    // 应用缩放
    width = width * exportSettings.scale
    height = height * exportSettings.scale
    
    // 确保尺寸有效
    if (width <= 0 || height <= 0) {
      width = 300
      height = 150
    }

    // 创建Canvas元素
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')
    canvas.width = width
    canvas.height = height
    
    // 如果是JPEG格式，设置背景色
    if (exportSettings.format === 'jpeg') {
      ctx.fillStyle = exportSettings.backgroundColor
      ctx.fillRect(0, 0, width, height)
    }

    // 将SVG转换为Blob URL
    // 添加宽高属性确保正确渲染
    let svgData = svgElement.outerHTML
    // 确保SVG有width和height属性
    if (!svgElement.hasAttribute('width')) {
      svgData = svgData.replace('<svg', `<svg width="${width}"`)
    }
    if (!svgElement.hasAttribute('height')) {
      svgData = svgData.replace('<svg', `<svg height="${height}"`)
    }
    
    const svgBlob = new Blob([svgData], { type: 'image/svg+xml;charset=utf-8' })
    const url = URL.createObjectURL(svgBlob)

    // 创建Image对象
    const img = new Image()
    img.onload = () => {
      // 在Canvas上绘制图像
      ctx.drawImage(img, 0, 0, width, height)
      // 将Canvas转换为指定格式
      canvas.toBlob((blob) => {
        saveAs(blob, `${exportSettings.filename}.${exportSettings.format}`)
        ElMessage.success('图片保存成功')
      }, `image/${exportSettings.format}`, exportSettings.format === 'jpeg' ? 0.9 : 1)
      URL.revokeObjectURL(url)
    }
    img.src = url
  } catch (error) {
    ElMessage.error('保存图片失败')
    console.error(error)
  }
}

// 提供一个示例SVG代码
onMounted(() => {
  svgCode.value = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 100 100" width="100" height="100">
  <circle cx="50" cy="50" r="40" stroke="black" stroke-width="2" fill="red" />
  <text x="50" y="55" font-size="12" text-anchor="middle" fill="white">SVG</text>
</svg>`
  updatePreview()
})
</script>

<style scoped>
.container {
  padding: 20px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.preview-container {
  min-height: 200px;
  border: 1px dashed #dcdfe6;
  border-radius: 4px;
  padding: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.preview-container svg {
  max-width: 100%;
  max-height: 100%;
}

.error-message {
  margin-top: 10px;
}
</style>
<template>
  <div class="app-container">
    <div class="warehouse-selector">
      <WarehouseSelector @change="handleWarehouseChange" />
    </div>
    <div class="stats-container">
      <div class="stats-panel">
        <div class="chart-card">
          <h3>仓库容量</h3>
          <div class="chart-container" ref="capacityChartRef"></div>
        </div>
        <div class="chart-card">
          <h3>货物分类</h3>
          <div class="chart-container" ref="categoryChartRef"></div>
        </div>
        <div class="chart-card">
          <h3>库存预警</h3>
          <div class="chart-container" ref="warningChartRef"></div>
        </div>
      </div>
      <div class="scene-container" ref="sceneContainer"></div>
      <div class="stats-panel">
        <div class="chart-card">
          <h3>出入库趋势</h3>
          <div class="chart-container" ref="trendChartRef"></div>
        </div>
        <div class="chart-card">
          <h3>库存周转率</h3>
          <div class="chart-container" ref="turnoverChartRef"></div>
        </div>
        <div class="chart-card">
          <h3>温湿度监控</h3>
          <div class="chart-container" ref="environmentChartRef"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import * as echarts from 'echarts'

// 货物类别列表
const categories = ['原材料', '成品', '半成品', '包装材料']

// 货物名称列表
const goodsNames = {
  '原材料': ['钢材', '铝材', '塑料粒子', '木材', '布料'],
  '成品': ['电视机', '冰箱', '洗衣机', '空调', '微波炉'],
  '半成品': ['主板', '显示屏', '电机', '压缩机', '控制器'],
  '包装材料': ['纸箱', '泡沫', '塑料袋', '胶带', '标签']
}

// 随机获取货物类别
const getRandomCategory = () => {
  return categories[Math.floor(Math.random() * categories.length)]
}

// 随机获取货物名称
const getRandomGoodsName = () => {
  const category = getRandomCategory()
  const names = goodsNames[category]
  return names[Math.floor(Math.random() * names.length)]
}

const sceneContainer = ref(null)
let scene, camera, renderer, controls
const shelves = ref([])

// 货架参数
const SHELF_WIDTH = 2
const SHELF_HEIGHT = 3
const SHELF_DEPTH = 1
const GAP = 2 // 增加走道宽度
const ROWS = ref(10)
const COLS = ref(10)

// 创建tooltip
const tooltip = document.createElement('div')
tooltip.style.position = 'absolute'
tooltip.style.backgroundColor = 'rgba(26, 41, 64, 0.95)'
tooltip.style.color = 'white'
tooltip.style.padding = '15px'
tooltip.style.borderRadius = '8px'
tooltip.style.display = 'none'
tooltip.style.pointerEvents = 'none'
tooltip.style.zIndex = '1000'
tooltip.style.boxShadow = '0 4px 12px rgba(0, 0, 0, 0.15)'
tooltip.style.backdropFilter = 'blur(4px)'
tooltip.style.border = '1px solid rgba(255, 255, 255, 0.1)'
tooltip.style.fontSize = '14px'
tooltip.style.lineHeight = '1.5'
tooltip.style.minWidth = '200px'

// 鼠标移动事件处理
const onMouseMove = (event) => {
  const rect = renderer.domElement.getBoundingClientRect()
  const mouse = new THREE.Vector2(
    ((event.clientX - rect.left) / rect.width) * 2 - 1,
    -((event.clientY - rect.top) / rect.height) * 2 + 1
  )

  const raycaster = new THREE.Raycaster()
  raycaster.setFromCamera(mouse, camera)

  const intersects = raycaster.intersectObjects(shelves.value, true)

  if (intersects.length > 0) {
    let shelf = intersects[0].object
    while (shelf && !shelf.userData.hasOwnProperty('capacity')) {
      shelf = shelf.parent
    }

    if (shelf && shelf.userData) {
      const capacity = shelf.userData.capacity
      const row = shelf.userData.row
      const col = shelf.userData.col
      const shelfId = `${String.fromCharCode(65 + row)}${col + 1}`
      const category = getRandomCategory()
      const goodsName = getRandomGoodsName()
      const stockQuantity = Math.round(capacity * 100)
      const status = capacity > 0.8 ? '库存充足' : capacity > 0.3 ? '库存正常' : '库存不足'
      const statusColor = capacity > 0.8 ? '#67C23A' : capacity > 0.3 ? '#409EFF' : '#F56C6C'

      tooltip.style.display = 'block'
      tooltip.style.left = `${event.clientX + 15}px`
      tooltip.style.top = `${event.clientY + 15}px`
      tooltip.innerHTML = `
        <div style="margin-bottom: 8px; font-weight: bold; font-size: 16px;">货架 ${shelfId}</div>
        <div style="display: flex; justify-content: space-between; margin-bottom: 4px;">
          <span>货架位置:</span>
          <span>${row + 1}排${col + 1}列</span>
        </div>
        <div style="display: flex; justify-content: space-between; margin-bottom: 4px;">
          <span>使用状态:</span>
          <span style="color: ${statusColor}">${status} (${Math.round(capacity * 100)}%)</span>
        </div>
        <div style="display: flex; justify-content: space-between; margin-bottom: 4px;">
          <span>货物类别:</span>
          <span>${category}</span>
        </div>
        <div style="display: flex; justify-content: space-between; margin-bottom: 4px;">
          <span>货物名称:</span>
          <span>${goodsName}</span>
        </div>
        <div style="display: flex; justify-content: space-between;">
          <span>库存数量:</span>
          <span>${stockQuantity}件</span>
        </div>
      `
    } else {
      tooltip.style.display = 'none'
    }
  } else {
    tooltip.style.display = 'none'
  }
}

// 创建货架
const createShelf = (row, col, capacity, totalRows, totalCols) => {
  const geometry = new THREE.BoxGeometry(SHELF_WIDTH, SHELF_HEIGHT, SHELF_DEPTH)
  const material = new THREE.MeshStandardMaterial({
    color: new THREE.Color().setHSL(0.3 - capacity * 0.3, 0.8, 0.5) // 从绿色(0.3)渐变到红色(0)
  })
  const shelf = new THREE.Mesh(geometry, material)
  shelf.position.y = SHELF_HEIGHT / 2
  shelf.castShadow = true
  shelf.receiveShadow = true

  shelf.userData = { capacity, row, col }

  return shelf
}

// 初始化场景
const initScene = () => {
  scene = new THREE.Scene()
  scene.background = new THREE.Color(0x050d1a) // 更深的背景色，更容易与地面区分

  // 设置相机
  camera = new THREE.PerspectiveCamera(
    75,
    sceneContainer.value.clientWidth / sceneContainer.value.clientHeight,
    0.1,
    1000
  )
  camera.position.set(15, 15, 15)

  // 设置渲染器
  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(
    sceneContainer.value.clientWidth,
    sceneContainer.value.clientHeight
  )
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap
  sceneContainer.value.appendChild(renderer.domElement)

  // 设置轨道控制器
  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.05
  controls.maxPolarAngle = Math.PI / 2.5 // 限制相机最大俯角
  controls.minPolarAngle = Math.PI / 6 // 限制相机最小俯角
  controls.target.set(0, 0, 0) // 设置控制器目标点
  controls.update()

  // 添加地面
  const groundGeometry = new THREE.PlaneGeometry(100, 100)
  const groundMaterial = new THREE.MeshStandardMaterial({
    color: 0x1a2940,
    roughness: 0.6,
    metalness: 0.4
  })
  const ground = new THREE.Mesh(groundGeometry, groundMaterial)
  ground.rotation.x = -Math.PI / 2
  ground.receiveShadow = true
  scene.add(ground)

  // 添加网格
  const gridHelper = new THREE.GridHelper(100, 100, 0x4d9fff, 0x2c6af3)
  gridHelper.position.y = 0.01
  scene.add(gridHelper)

  // 优化光照
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4) // 减弱环境光
  scene.add(ambientLight)
  
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.6)
  directionalLight.position.set(20, 30, 20)
  directionalLight.castShadow = true // 启用阴影
  directionalLight.shadow.mapSize.width = 2048
  directionalLight.shadow.mapSize.height = 2048
  scene.add(directionalLight)

  // 添加辅助光源
  const pointLight = new THREE.PointLight(0xffffff, 0.3)
  pointLight.position.set(-20, 20, -20)
  scene.add(pointLight)

  // 创建初始货架布局
  const centerOffsetX = (COLS.value * (SHELF_WIDTH + GAP)) / 2
  const centerOffsetZ = (ROWS.value * (SHELF_DEPTH + GAP)) / 2

  for (let row = 0; row < ROWS.value; row++) {
    for (let col = 0; col < COLS.value; col++) {
      const capacity = Math.random()
      const shelf = createShelf(row, col, capacity, ROWS.value, COLS.value)
      shelf.position.x = col * (SHELF_WIDTH + GAP) - centerOffsetX + SHELF_WIDTH / 2
      shelf.position.z = row * (SHELF_DEPTH + GAP) - centerOffsetZ + SHELF_DEPTH / 2
      shelves.value.push(shelf)
      scene.add(shelf)
    }
  }

  // 添加tooltip到DOM
  document.body.appendChild(tooltip)

  // 添加鼠标移动事件监听
  renderer.domElement.addEventListener('mousemove', onMouseMove)

  animate()
}

// 动画循环
const animate = () => {
  requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

const capacityChartRef = ref(null)
const categoryChartRef = ref(null)
const warningChartRef = ref(null)
const trendChartRef = ref(null)
const turnoverChartRef = ref(null)
const environmentChartRef = ref(null)
let capacityChart = null
let categoryChart = null
let warningChart = null
let trendChart = null
let turnoverChart = null
let environmentChart = null

// 初始化货物分类图表
const initCategoryChart = () => {
  if (!categoryChartRef.value) return
  categoryChart = echarts.init(categoryChartRef.value)
  
  const option = {
    series: [{
      type: 'pie',
      radius: '50%',
      data: [
        { value: 1048, name: '原材料', itemStyle: { color: '#91CC75' } },
        { value: 735, name: '成品', itemStyle: { color: '#FAC858' } },
        { value: 580, name: '半成品', itemStyle: { color: '#EE6666' } },
        { value: 484, name: '包装材料', itemStyle: { color: '#73C0DE' } }
      ],
      emphasis: {
        itemStyle: {
          shadowBlur: 10,
          shadowOffsetX: 0,
          shadowColor: 'rgba(0, 0, 0, 0.5)'
        }
      }
    }]
  }
  
  categoryChart.setOption(option)
}

// 初始化库存预警图表
const initWarningChart = () => {
  if (!warningChartRef.value) return
  warningChart = echarts.init(warningChartRef.value)
  
  const option = {
    tooltip: {
      trigger: 'item'
    },
    legend: {
      top: '5%',
      left: 'center'
    },
    series: [
      {
        type: 'pie',
        radius: ['40%', '70%'],
        avoidLabelOverlap: false,
        itemStyle: {
          borderRadius: 10,
          borderColor: '#fff',
          borderWidth: 2
        },
        label: {
          show: false,
          position: 'center'
        },
        emphasis: {
          label: {
            show: true,
            fontSize: '14',
            fontWeight: 'bold'
          }
        },
        labelLine: {
          show: false
        },
        data: [
          { value: 5, name: '库存不足', itemStyle: { color: '#F56C6C' } },
          { value: 3, name: '即将过期', itemStyle: { color: '#E6A23C' } },
          { value: 2, name: '库存积压', itemStyle: { color: '#409EFF' } }
        ]
      }
    ]
  }
  
  warningChart.setOption(option)
}

// 初始化库存周转率图表
const initTurnoverChart = () => {
  if (!turnoverChartRef.value) return
  turnoverChart = echarts.init(turnoverChartRef.value)
  
  const option = {
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'shadow'
      }
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      containLabel: true
    },
    xAxis: [
      {
        type: 'category',
        data: ['1月', '2月', '3月', '4月', '5月', '6月'],
        axisTick: {
          alignWithLabel: true
        }
      }
    ],
    yAxis: [
      {
        type: 'value',
        name: '周转率',
        axisLabel: {
          formatter: '{value}次/月'
        }
      }
    ],
    series: [
      {
        name: '周转率',
        type: 'bar',
        barWidth: '60%',
        data: [3.2, 3.5, 3.7, 3.4, 3.8, 3.6],
        itemStyle: {
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: '#83bff6' },
            { offset: 0.5, color: '#188df0' },
            { offset: 1, color: '#188df0' }
          ])
        }
      }
    ]
  }
  
  turnoverChart.setOption(option)
}

// 初始化温湿度监控图表
const initEnvironmentChart = () => {
  if (!environmentChartRef.value) return
  environmentChart = echarts.init(environmentChartRef.value)
  
  const option = {
    grid: {
      top: '10%',
      left: '3%',
      right: '4%',
      bottom: '3%',
      containLabel: true
    },
    tooltip: {
      trigger: 'axis',
      formatter: '{b}<br/>{a0}: {c0}°C<br/>{a1}: {c1}%'
    },
    xAxis: {
      type: 'category',
      boundaryGap: false,
      data: ['00:00', '04:00', '08:00', '12:00', '16:00', '20:00', '24:00']
    },
    yAxis: [
      {
        type: 'value',
        name: '温度(°C)',
        position: 'left',
        axisLabel: {
          formatter: '{value}°C'
        }
      },
      {
        type: 'value',
        name: '湿度(%)',
        position: 'right',
        axisLabel: {
          formatter: '{value}%'
        }
      }
    ],
    series: [
      {
        name: '温度',
        type: 'line',
        smooth: true,
        data: [20, 21, 22, 23, 24, 23, 22]
      },
      {
        name: '湿度',
        type: 'line',
        smooth: true,
        yAxisIndex: 1,
        data: [55, 54, 53, 52, 53, 54, 55]
      }
    ]
  }
  
  environmentChart.setOption(option)
}

// 初始化仓库容量图表
const initCapacityChart = () => {
  if (!capacityChartRef.value) return
  capacityChart = echarts.init(capacityChartRef.value)
  
  const option = {
    tooltip: {
      trigger: 'item',
      formatter: '{b}: {c}%'
    },
    series: [{
      type: 'gauge',
      startAngle: 180,
      endAngle: 0,
      min: 0,
      max: 100,
      splitNumber: 10,
      radius: '100%',
      itemStyle: {
        color: '#5470C6'
      },
      progress: {
        show: true,
        roundCap: true,
        width: 18
      },
      pointer: {
        show: false
      },
      axisLine: {
        roundCap: true,
        lineStyle: {
          width: 18
        }
      },
      axisTick: {
        show: false
      },
      splitLine: {
        show: false
      },
      axisLabel: {
        show: false
      },
      title: {
        show: false
      },
      detail: {
        offsetCenter: [0, '5%'],
        fontSize: 24,
        fontWeight: 'bold',
        formatter: '{value}%',
        color: '#fff'
      },
      data: [{
        value: 78,
        name: '使用率'
      }]
    }]
  }
  
  capacityChart.setOption(option)
}

// 初始化出入库趋势图表
const initTrendChart = () => {
  if (!trendChartRef.value) return
  trendChart = echarts.init(trendChartRef.value)
  
  const option = {
    tooltip: {
      trigger: 'axis',
      axisPointer: {
        type: 'cross',
        label: {
          backgroundColor: '#6a7985'
        }
      }
    },
    legend: {
      data: ['入库量', '出库量'],
      textStyle: {
        color: '#fff'
      }
    },
    grid: {
      left: '3%',
      right: '4%',
      bottom: '3%',
      containLabel: true
    },
    xAxis: [
      {
        type: 'category',
        boundaryGap: false,
        data: ['周一', '周二', '周三', '周四', '周五', '周六', '周日'],
        axisLabel: {
          color: '#fff'
        }
      }
    ],
    yAxis: [
      {
        type: 'value',
        axisLabel: {
          color: '#fff',
          formatter: '{value}件'
        }
      }
    ],
    series: [
      {
        name: '入库量',
        type: 'line',
        stack: 'Total',
        areaStyle: {},
        emphasis: {
          focus: 'series'
        },
        data: [120, 132, 101, 134, 90, 230, 210]
      },
      {
        name: '出库量',
        type: 'line',
        stack: 'Total',
        areaStyle: {},
        emphasis: {
          focus: 'series'
        },
        data: [220, 182, 191, 234, 290, 330, 310]
      }
    ]
  }
  
  trendChart.setOption(option)
}

// 在组件挂载时初始化图表
onMounted(() => {
  initScene()
  initCapacityChart()
  initCategoryChart()
  initWarningChart()
  initTrendChart()
  initTurnoverChart()
  initEnvironmentChart()
})
</script>

<style scoped>
.app-container {
  width: 100%;
  height: 100vh;
  background-color: #0a192f;
  padding: 20px;
  box-sizing: border-box;
}

.warehouse-selector {
  margin-bottom: 20px;
}

.stats-container {
  display: flex;
  gap: 20px;
  height: calc(100% - 60px);
  padding: 0 20px;
}

.stats-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 20px;
  max-width: 400px;
}

.chart-card {
  flex: 1;
  background-color: rgba(26, 41, 64, 0.95);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(4px);
}

.chart-card h3 {
  color: #ffffff;
  margin: 0 0 15px 0;
  font-size: 16px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 8px;
}

.chart-container {
  height: calc(100% - 31px);
  width: 100%;
}

.scene-container {
  flex: 2;
  background-color: rgba(26, 41, 64, 0.95);
  border-radius: 12px;
  overflow: hidden;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  border: 1px solid rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(4px);
}
</style>
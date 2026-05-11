<template>
  <div class="keyboard-editor">
    <!-- Тулбар -->
    <header class="toolbar">
      <button @click="exportJSON" class="btn primary"> Экспорт JSON</button>
      <label class="btn secondary">
         Импорт JSON
        <input type="file" accept=".json" hidden @change="importJSON" />
      </label>
      <button @click="loadDefaultLayout" class="btn"> Сброс раскладки</button>
      <label class="btn outline">
         Загрузить .glb модель
        <input type="file" accept=".glb,.gltf" hidden @change="loadModel" />
      </label>
      <span class="status">{{ status }}</span>
    </header>

    <!-- 3D Viewport -->
    <div class="viewport" ref="viewportRef" @mousemove="onMouseMove" @click="onClick"></div>

    <!-- Панель свойств -->
    <aside class="sidebar" v-if="selectedKey">
      <h3>🔧 Свойства клавиши</h3>
      <div class="prop-group">
        <label>Надпись:</label>
        <input v-model="selectedKey.label" @input="updateSelectedKey" />
      </div>
      <div class="prop-group">
        <label>Код (keycode):</label>
        <input v-model="selectedKey.code" @input="updateSelectedKey" />
      </div>
      <div class="prop-group">
        <label>Цвет корпуса:</label>
        <input type="color" v-model="selectedKey.color" @input="updateSelectedKey" />
      </div>
      <div class="prop-group">
        <label>Цвет подсветки:</label>
        <input type="color" v-model="selectedKey.emissive" @input="updateSelectedKey" />
      </div>
      <button @click="selectedKey = null" class="btn danger">Закрыть</button>
    </aside>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js'

//Состояние 
const viewportRef = ref(null)
const selectedKey = ref(null)
const status = ref('Готово')
let mode = 'layout' // 'layout' | 'model'

// Базовая раскладка
const defaultLayout = [
  { id: 'esc', x: 0, y: 0, w: 1, h: 1, label: 'Esc', code: 'Escape', color: '#2a2a35', emissive: '#000000' },
  { id: '1', x: 1.25, y: 0, w: 1, h: 1, label: '1', code: 'Digit1', color: '#2a2a35', emissive: '#000000' },
  { id: 'tab', x: 0, y: 1.25, w: 1.5, h: 1, label: 'Tab', code: 'Tab', color: '#2a2a35', emissive: '#000000' },
  { id: 'q', x: 1.75, y: 1.25, w: 1, h: 1, label: 'Q', code: 'KeyQ', color: '#2a2a35', emissive: '#000000' },
  { id: 'w', x: 3, y: 1.25, w: 1, h: 1, label: 'W', code: 'KeyW', color: '#2a2a35', emissive: '#000000' },
  { id: 'space', x: 1.25, y: 2.5, w: 6, h: 1, label: 'Space', code: 'Space', color: '#2a2a35', emissive: '#000000' }
]

const layout = reactive(JSON.parse(JSON.stringify(defaultLayout)))

//переменные
let scene, camera, renderer, controls, raycaster, mouse
let keyMeshes = []
let loadedModelGroup = null
let animationId
const hoveredMesh = ref(null)
const textureCache = new Map() // кэш для CanvasTexture

// Утилиты
function createKeyTexture(label, color, width = 256, height = 256) {
  const cacheKey = `${label}|${color}|${width}x${height}`
  if (textureCache.has(cacheKey)) return textureCache.get(cacheKey)

  const canvas = document.createElement('canvas')
  canvas.width = width
  canvas.height = height
  const ctx = canvas.getContext('2d')
  
  // Фон
  ctx.fillStyle = color
  ctx.fillRect(0, 0, width, height)
  
  // Тень/обводка
  ctx.strokeStyle = 'rgba(0,0,0,0.3)'
  ctx.lineWidth = 4
  ctx.strokeRect(2, 2, width - 4, height - 4)

  // Текст
  ctx.fillStyle = '#ffffff'
  ctx.font = 'bold 48px system-ui, sans-serif'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText(label.toUpperCase(), width / 2, height / 2)

  const tex = new THREE.CanvasTexture(canvas)
  tex.needsUpdate = true
  textureCache.set(cacheKey, tex)
  return tex
}

function disposeTexture(tex) {
  if (tex?.image) tex.dispose()
}

// Инициализация
onMounted(() => {
  const el = viewportRef.value
  if (!el) return

  scene = new THREE.Scene()
  scene.background = new THREE.Color('#0b0b14')

  camera = new THREE.PerspectiveCamera(45, el.clientWidth / el.clientHeight, 0.1, 100)
  camera.position.set(0, 10, 10)

  renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
  renderer.setSize(el.clientWidth, el.clientHeight)
  el.appendChild(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.target.set(0, 0, 0)

  //Cвет
  scene.add(new THREE.AmbientLight(0xffffff, 0.6))
  const dir = new THREE.DirectionalLight(0xffffff, 0.8)
  dir.position.set(5, 10, 5)
  scene.add(dir)

  raycaster = new THREE.Raycaster()
  mouse = new THREE.Vector2()

  //Ресайз
  const ro = new ResizeObserver(() => {
    const w = el.clientWidth, h = el.clientHeight
    camera.aspect = w / h
    camera.updateProjectionMatrix()
    renderer.setSize(w, h)
  })
  ro.observe(el)

  buildLayout()
  animate()
})

//Очистка
onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
  controls?.dispose()
  renderer?.dispose()
  keyMeshes.forEach(m => {
    m.geometry?.dispose()
    disposeTexture(m.material.map)
    m.material?.dispose()
  })
  textureCache.forEach(tex => tex.dispose())
  textureCache.clear()
})

//Логика сцены в моих глазах
function animate() {
  animationId = requestAnimationFrame(animate)
  controls.update()
  renderer.render(scene, camera)
}

function clearScene() {
  keyMeshes.forEach(m => {
    scene.remove(m)
    m.geometry?.dispose()
    disposeTexture(m.material.map)
    m.material?.dispose()
  })
  keyMeshes = []
  if (loadedModelGroup) {
    scene.remove(loadedModelGroup)
    loadedModelGroup.traverse(c => c.geometry?.dispose())
    loadedModelGroup = null
  }
}

function buildLayout() {
  clearScene()
  mode = 'layout'
  const unit = 1.9

  layout.forEach(k => {
    const geom = new THREE.BoxGeometry(k.w * unit, 0.4, k.h * unit)
    const tex = createKeyTexture(k.label, k.color)
    const mat = new THREE.MeshStandardMaterial({
      map: tex,
      color: 0xffffff,
      roughness: 0.6,
      metalness: 0.1,
      emissive: new THREE.Color(k.emissive || '#000000'),
      emissiveIntensity: k.emissive && k.emissive !== '#000000' ? 0.8 : 0
    })
    const mesh = new THREE.Mesh(geom, mat)
    mesh.position.set(
      (k.x + k.w / 2) * unit - 4,
      0.2,
      (k.y + k.h / 2) * unit - 2
    )
    mesh.userData = { ...k, isKey: true, originalColor: k.color }
    scene.add(mesh)
    keyMeshes.push(mesh)
  })
  status.value = `Раскладка: ${layout.length} клавиш`
}

//Загрузка модели
function loadModel(e) {
  const file = e.target.files[0]
  if (!file) return
  status.value = 'Загрузка модели...'
  
  const url = URL.createObjectURL(file)
  const loader = new GLTFLoader()
  loader.load(url, (gltf) => {
    clearScene()
    loadedModelGroup = gltf.scene
    mode = 'model'
    scene.add(loadedModelGroup)
    status.value = 'Модель загружена'
    URL.revokeObjectURL(url)
  }, undefined, (err) => {
    console.error(err)
    status.value = 'Ошибка загрузки'
  })
}

// Взаимодействие
function getIntersectedKey(e) {
  const rect = viewportRef.value.getBoundingClientRect()
  mouse.x = ((e.clientX - rect.left) / rect.width) * 2 - 1
  mouse.y = -((e.clientY - rect.top) / rect.height) * 2 + 1
  raycaster.setFromCamera(mouse, camera)
  const targets = mode === 'layout' ? keyMeshes : (loadedModelGroup?.children.filter(c => c.isMesh) || [])
  const hits = raycaster.intersectObjects(targets, true)
  return hits.length ? hits[0].object : null
}

function onMouseMove(e) {
  const mesh = getIntersectedKey(e)
  if (mesh !== hoveredMesh.value) {
    if (hoveredMesh.value && hoveredMesh.value !== selectedKey.value) {
      hoveredMesh.value.material.emissive.set(hoveredMesh.value.userData.originalColor || '#000000')
      hoveredMesh.value.material.emissiveIntensity = 0
    }
    hoveredMesh.value = mesh
    if (mesh) {
      document.body.style.cursor = 'pointer'
    } else {
      document.body.style.cursor = 'default'
    }
  }
}

function onClick(e) {
  const mesh = getIntersectedKey(e)
  if (mesh) {
    selectedKey.value = mesh.userData
    mesh.material.emissive.set('#ffaa00')
    mesh.material.emissiveIntensity = 0.4
    status.value = `Выбрано: ${mesh.userData.id || 'Mesh'}`
  } else {
    if (selectedKey.value) {
      const prev = keyMeshes.find(m => m.userData.id === selectedKey.value.id)
      if (prev) {
        prev.material.emissive.set(selectedKey.value.emissive || '#000000')
        prev.material.emissiveIntensity = selectedKey.value.emissive !== '#000000' ? 0.8 : 0
      }
    }
    selectedKey.value = null
    status.value = 'Готово'
  }
}

function updateSelectedKey() {
  if (!selectedKey.value) return
  const mesh = keyMeshes.find(m => m.userData.id === selectedKey.value.id)
  if (!mesh) return

  // Обновляем реактивные данные
  const item = layout.find(k => k.id === selectedKey.value.id)
  if (item) Object.assign(item, selectedKey.value)

  // Обновляем 3D объект
  mesh.userData = { ...selectedKey.value, isKey: true, originalColor: selectedKey.value.color }
  mesh.material.color.set(0xffffff) // цвет задается в map
  mesh.material.emissive.set(selectedKey.value.emissive)
  mesh.material.emissiveIntensity = selectedKey.value.emissive !== '#000000' ? 0.8 : 0
  
  // Пересоздаем текстуру
  disposeTexture(mesh.material.map)
  const newTex = createKeyTexture(selectedKey.value.label, selectedKey.value.color)
  mesh.material.map = newTex
  mesh.material.needsUpdate = true
}

// Экспорт/Импорт
function exportJSON() {
  const data = JSON.stringify(layout, null, 2)
  const blob = new Blob([data], { type: 'application/json' })
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = `keyboard-layout-${Date.now()}.json`
  a.click()
  URL.revokeObjectURL(url)
  status.value = 'Экспорт сохранен'
}

function importJSON(e) {
  const file = e.target.files[0]
  if (!file) return
  const reader = new FileReader()
  reader.onload = (ev) => {
    try {
      const parsed = JSON.parse(ev.target.result)
      if (Array.isArray(parsed) && parsed.length && parsed[0].x !== undefined) {
        layout.length = 0
        parsed.forEach(k => layout.push(k))
        buildLayout()
        selectedKey.value = null
        status.value = 'Раскладка импортирована'
      } else {
        throw new Error('Неверный формат')
      }
    } catch (err) {
      status.value = 'Ошибка импорта: ' + err.message
    }
  }
  reader.readAsText(file)
}

function loadDefaultLayout() {
  layout.length = 0
  defaultLayout.forEach(k => layout.push({ ...k }))
  buildLayout()
  selectedKey.value = null
  status.value = 'Раскладка сброшена'
}
</script>

<style scoped>
.keyboard-editor {
  display: grid;
  grid-template-columns: 1fr 260px;
  height: 100vh;
  background: #0b0b14;
  color: #e2e8f0;
  font-family: system-ui, -apple-system, sans-serif;
}
.toolbar {
  grid-column: 1 / -1;
  padding: 10px 16px;
  background: #111827;
  border-bottom: 1px solid #1f2937;
  display: flex;
  gap: 8px;
  align-items: center;
}
.btn {
  padding: 6px 12px;
  border-radius: 6px;
  border: 1px solid #374151;
  background: #1f2937;
  color: #e2e8f0;
  cursor: pointer;
  font-size: 13px;
  display: inline-flex;
  align-items: center;
}
.btn:hover { background: #374151; }
.primary { background: #2563eb; border-color: #1d4ed8; }
.primary:hover { background: #1d4ed8; }
.secondary { background: #059669; border-color: #047857; }
.outline { background: transparent; border: 1px dashed #4b5563; }
.danger { background: #dc2626; border-color: #b91c1c; margin-top: auto; }
.status { margin-left: auto; color: #9ca3af; font-size: 12px; }

.viewport {
  position: relative;
  background: radial-gradient(circle at center, #151525 0%, #0b0b14 100%);
}
.sidebar {
  background: #111827;
  padding: 16px;
  border-left: 1px solid #1f2937;
  display: flex;
  flex-direction: column;
  gap: 14px;
}
.sidebar h3 { margin: 0; font-size: 16px; }
.prop-group { display: flex; flex-direction: column; gap: 4px; }
.prop-group label { font-size: 12px; color: #9ca3af; }
.prop-group input {
  background: #1f2937;
  border: 1px solid #374151;
  color: #f3f4f6;
  padding: 6px 8px;
  border-radius: 4px;
  font-size: 13px;
}
input[type="color"] { padding: 2px; height: 32px; cursor: pointer; }
</style>
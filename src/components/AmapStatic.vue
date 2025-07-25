<template>
  <div class="main-container">
    <div class="map-section" ref="mapContainer"></div>
    <div class="photos-section">
      <div class="photos-list">
        <div v-if="sortedClusterPhotos.length === 0" style="color: #aaa; text-align: center; width: 100%; padding: 40px 0;">请选择地图上的照片点</div>
        <div v-for="photo in sortedClusterPhotos" :key="photo.url" class="photo-item" v-else>
          <img :src="photo.url" @click="openPreview(photo)" />
          <div class="photo-date">{{ formatDate(photo.date) }}</div>
        </div>
      </div>
    </div>
    <!-- 大图预览弹窗 -->
    <div v-if="previewPhoto" class="photo-preview-overlay" @click.self="closePreview">
      <div class="photo-preview-content">
        <transition name="fade">
          <div v-if="previewTip" class="photo-preview-tip">{{ previewTip }}</div>
        </transition>
        <img :src="previewPhoto.url" class="photo-preview-img" @wheel="onPreviewWheel" />
        <button class="photo-preview-close" @click="closePreview">×</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import * as exifr from 'exifr'
import { computed } from 'vue'
import { ref as vueRef } from 'vue'

function formatDate(date) {
  if (!date) return ''
  const d = new Date(date)
  return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')} ${String(d.getHours()).padStart(2,'0')}:${String(d.getMinutes()).padStart(2,'0')}`
}
// 获取所有图片路径
const photoModules = import.meta.glob('../assets/photos/*.{jpg,JPEG,png,PNG}', {eager: true});
const photoList = Object.entries(photoModules).map(([path, mod]) => {
  return {
    url: mod.default, // 这是 Vite 处理后的图片 URL
    title: path.split('/').pop()
  }
});

const mapContainer = ref(null)
const amapKey = '7660525720c1081f9a29d0d35754bc31'
const photos = ref([])
const selectedClusterPhotos = ref([])
// 新增：计算属性，按时间升序排列
const sortedClusterPhotos = computed(() => {
  if (!selectedClusterPhotos.value.length) return []
  return [...selectedClusterPhotos.value].sort((a, b) => {
    if (!a.date) return -1
    if (!b.date) return 1
    return new Date(a.date) - new Date(b.date)
  })
})
async function extractPhotos() {
  const arr = []
  for (const photo of photoList) {
    const response = await fetch(photo.url)
    const blob = await response.blob()
    const gps = await exifr.gps(blob)
    const exif = await exifr.parse(blob, ['DateTimeOriginal'])
    arr.push({
      lng: gps?.longitude ?? null,
      lat: gps?.latitude ?? null,
      url: photo.url,
      title: photo.title,
      date: exif?.DateTimeOriginal ? new Date(exif.DateTimeOriginal) : null
    })
  }
  photos.value = arr
}


function loadAmapScript() {
  return new Promise((resolve, reject) => {
    if (window.AMap && window.AMap.MarkerClusterer) {
      resolve()
      return
    }
    const script = document.createElement('script')
    script.src = `https://webapi.amap.com/maps?v=1.4.15&key=${amapKey}&plugin=AMap.MarkerClusterer`
    script.async = true
    script.onload = () => {
      const check = () => {
        if (window.AMap && window.AMap.MarkerClusterer) {
          resolve()
        } else {
          setTimeout(check, 50)
        }
      }
      check()
    }
    script.onerror = () => reject('高德地图脚本加载失败，请检查网络和key')
    document.head.appendChild(script)
  })
}

const previewPhoto = ref(null)
const previewTip = vueRef('')
let tipTimer = null
function showPreviewTip(msg) {
  previewTip.value = msg
  if (tipTimer) clearTimeout(tipTimer)
  tipTimer = setTimeout(() => { previewTip.value = '' }, 1200)
}

function openPreview(photo) {
  previewPhoto.value = photo
}
function closePreview() {
  previewPhoto.value = null
}

function onPreviewWheel(e) {
  e.preventDefault();
  if (!previewPhoto.value) return;
  const idx = sortedClusterPhotos.value.findIndex(p => p.url === previewPhoto.value.url);
  if (e.deltaY > 0) {
    if (idx < sortedClusterPhotos.value.length - 1) {
      previewPhoto.value = sortedClusterPhotos.value[idx + 1];
    } else {
      showPreviewTip('到底啦')
    }
  } else if (e.deltaY < 0) {
    if (idx > 0) {
      previewPhoto.value = sortedClusterPhotos.value[idx - 1];
    } else {
      showPreviewTip('到顶啦')
    }
  }
}

onMounted(async () => {
  await extractPhotos()
  await loadAmapScript()
  const map = new window.AMap.Map(mapContainer.value, {
    center: [106.397428, 33.90923],
    zoom: 4
  })

  const markers = photos.value.map(photo => {
    return new window.AMap.Marker({
      position: [photo.lng, photo.lat],
      content: `<div style="border-radius:50%;overflow:hidden;width:40px;height:40px;border:2px solid #fff;box-shadow:0 2px 6px rgba(0,0,0,0.3);">
        <img src="${photo.url}" style="width:100%;height:100%;object-fit:cover;" />
      </div>`,
      anchor: 'center',
      extData: photo
    })
  })
  const cluster = new window.AMap.MarkerClusterer(map, markers, {
    gridSize: 80,
    zoomOnClick: false,
    renderClusterMarker: function(context) {
      const firstPhoto = context.markers[0].getExtData()

      const count = context.count
      context.marker.setContent(`
    <div style="
      position:relative;
      width:50px;
      height:50px;
      border-radius:50%;
      overflow:hidden;
      border:2px solid #fff;
      box-shadow:0 2px 6px rgba(0,0,0,0.15);
      background:#fff;
      display:flex;
      align-items:center;
      justify-content:center;
      ">
      <img src="${firstPhoto.url}" style="width:100%;height:100%;object-fit:cover;display:block;" />
      <span style="
        position:absolute;
        right:2px;
        bottom:2px;
        background:rgba(0,0,0,0);
        color:#fff;
        font-size:15px;
        padding:1px 7px;
        border-radius:8px 0 8px 0;
        z-index:2;
        font-weight:bold;
        ">
        ${count}
      </span>
    </div>
  `)
      context.marker.setAnchor('center')
    }
  })

  cluster.on('click', function (event) {
  selectedClusterPhotos.value = event.markers.map(marker => marker.getExtData())
})

    markers.forEach(marker => {
    marker.on('click', () => {
        const photo = marker.getExtData()
        selectedClusterPhotos.value = [photo]
    })
    })
})
</script>

<style scoped>
.main-container {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  width: 100vw;
  min-height: 800px;
  box-sizing: border-box;
  padding: 104px 40px 0 40px; /* header下40px，左右40px */
  gap: 40px; /* 地图与照片合集间距 */
}
.map-section {
  flex: 2 1 0;
  min-width: 400px;
  min-height: 60vh;
  height: 84vh;
  border-radius: 12px;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0,0,0,0.08);
}
.photos-section {
  flex: 1 1 0;
  min-width: 320px;
  max-width: 600px;
  height: 84vh;
  background: #fafbfc;
  border-radius: 12px;
  padding: 18px 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.06);
  display: flex;
  flex-direction: column;
  overflow-y: auto; /* 关键：只让右侧滚动 */
}
.photos-list {
  flex: 1 1 0;
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}
.photo-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  /* cursor: pointer;  // 移除 */
}
.photo-item img {
  width: 120px;
  height: 90px;
  object-fit: cover;
  border-radius: 10px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.08);
  transition: transform 0.18s cubic-bezier(.4,1.5,.5,1), box-shadow 0.18s;
  cursor: pointer; /* 只在图片上显示小手 */
}
.photo-item img:hover {
  transform: scale(1.08);
  box-shadow: 0 4px 16px rgba(0,0,0,0.16);
}
.photo-date {
  margin-top: 6px;
  color: #888;
  font-size: 13px;
  cursor: default;
}
/* 大图预览弹窗样式 */
.photo-preview-overlay {
  position: fixed;
  left: 0; top: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.55);
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
}
.photo-preview-content {
  position: relative;
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 4px 32px rgba(0,0,0,0.25);
  padding: 24px;
  display: flex;
  flex-direction: column;
  align-items: center;
  max-width: 90vw;
  max-height: 90vh;
}
.photo-preview-img {
  max-width: 70vw;
  max-height: 70vh;
  border-radius: 10px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.18);
}
.photo-preview-close {
  position: absolute;
  top: 8px;
  right: 12px;
  background: rgba(0,0,0,0.5);
  color: #fff;
  border: none;
  border-radius: 50%;
  width: 32px;
  height: 32px;
  font-size: 22px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}
.photo-preview-close:hover {
  background: rgba(0,0,0,0.8);
}
.photo-preview-tip {
  position: absolute;
  top: 12px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0,0,0,0.75);
  color: #fff;
  padding: 6px 18px;
  border-radius: 16px;
  font-size: 16px;
  z-index: 10;
  pointer-events: none;
  user-select: none;
  box-shadow: 0 2px 8px rgba(0,0,0,0.18);
}
.fade-enter-active, .fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from, .fade-leave-to {
  opacity: 0;
}
@media (max-width: 900px) {
  .main-container {
    flex-direction: column;
    padding: 40px 10px 0 10px;
    gap: 20px;
  }
  .map-section, .photos-section {
    width: 100%;
    min-width: 0;
    max-width: 100%;
    height: 40vh;
  }
}
</style>
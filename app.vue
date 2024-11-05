<template>
  <div class="map-container">
    <!-- 地図を表示するコンテナ -->
    <div ref="mapContainer" class="map-wrapper" />

    <!-- 地図スタイル切り替えパネル -->
    <div class="style-switcher">
      <!-- 地図スタイル切り替えボタン -->
      <button
        v-for="style in mapStyles"
        :key="style.id"
        @click="switchStyle(style)"
        :class="{ active: currentStyle === style.id }"
      >
        {{ style.name }}
      </button>
      
      <!-- Amazon Location Service選択時の国境線切り替えボタン -->
      <div v-if="currentStyle === 'amazon'" class="border-toggle">
        <button
          @click="toggleBorders"
          :class="{ active: !bordersHidden }"
        >
          {{ bordersHidden ? '国境線を表示' : '国境線を非表示' }}
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
// 必要なライブラリのインポート
import { onMounted, ref } from 'vue'
import { Amplify } from 'aws-amplify'
import { withIdentityPoolId } from '@aws/amazon-location-utilities-auth-helper'
import outputs from '../amplify_outputs.json'

// AWS Amplifyの設定
Amplify.configure(outputs)

// リアクティブな参照の定義
const mapContainer = ref(null)
const map = ref(null)
const currentStyle = ref('amazon')
const bordersHidden = ref(false)

// 環境変数と設定値の定義
const MAPTILER_KEY = import.meta.env.VITE_MAPTILER_KEY
const REGION = outputs.geo.aws_region
const MAP_NAME = outputs.geo.maps.default
const IDENTITY_POOL_ID = outputs.auth.identity_pool_id

// 地図スタイルの定義
const mapStyles = [
  {
    id: 'amazon',
    name: 'Amazon Location',
    isAmazonStyle: true,
    url: `https://maps.geo.${REGION}.amazonaws.com/maps/v0/maps/${MAP_NAME}/style-descriptor`
  },
  {
    id: 'osm',
    name: 'OpenStreetMap',
    url: {
      version: 8,
      sources: {
        'osm': {
          type: 'raster',
          tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'],
          tileSize: 256,
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }
      },
      layers: [{
        id: 'osm-tiles',
        type: 'raster',
        source: 'osm',
        minzoom: 0,
        maxzoom: 19
      }]
    }
  },
  {
    id: 'maptiler',
    name: 'MapTiler White',
    url: `https://api.maptiler.com/maps/90b754d8-eb36-477e-bc28-337bc7bfaaf1/style.json?key=${MAPTILER_KEY}`
  }
]

/**
 * 国境線の表示/非表示を切り替える関数
 */
const toggleBorders = () => {
  if (!map.value) return

  const layers = map.value.getStyle().layers
  layers.forEach(layer => {
    if (isBorderLayer(layer.id)) {
      map.value.setLayoutProperty(
        layer.id,
        'visibility',
        bordersHidden.value ? 'visible' : 'none'
      )
    }
  })
  bordersHidden.value = !bordersHidden.value
}

/**
 * レイヤーIDが国境線関連かどうかを判定
 */
const isBorderLayer = (layerId) => {
  return layerId.includes('border') || 
         layerId.includes('boundary') ||
         layerId.includes('admin')
}

/**
 * 地図スタイルを切り替える関数
 */
const switchStyle = async (style) => {
  try {
    if (!map.value) return

    // 現在の地図の状態を保存
    const mapState = {
      center: map.value.getCenter(),
      zoom: map.value.getZoom(),
      bearing: map.value.getBearing(),
      pitch: map.value.getPitch()
    }

    if (style.isAmazonStyle) {
      await switchToAmazonStyle(style, mapState)
    } else {
      await switchToOtherStyle(style, mapState)
    }

    currentStyle.value = style.id
  } catch (error) {
    console.error('Style switch error:', error)
  }
}

/**
 * Amazon Location Serviceスタイルに切り替える
 */
const switchToAmazonStyle = async (style, mapState) => {
  const authHelper = await withIdentityPoolId(IDENTITY_POOL_ID)
  const newStyle = {
    style: style.url,
    ...authHelper.getMapAuthenticationOptions()
  }
  map.value.setStyle(newStyle.style, newStyle)
  
  map.value.once('style.load', () => {
    applyMapState(mapState)
    if (bordersHidden.value) {
      hideAllBorders()
    }
  })
}

/**
 * その他のスタイルに切り替える
 */
const switchToOtherStyle = async (style, mapState) => {
  map.value.setStyle(style.url)
  map.value.once('style.load', () => {
    applyMapState(mapState)
  })
}

/**
 * 保存した地図の状態を適用
 */
const applyMapState = (state) => {
  map.value.setCenter(state.center)
  map.value.setZoom(state.zoom)
  map.value.setBearing(state.bearing)
  map.value.setPitch(state.pitch)
}

/**
 * すべての国境線を非表示にする
 */
const hideAllBorders = () => {
  const layers = map.value.getStyle().layers
  layers.forEach(layer => {
    if (isBorderLayer(layer.id)) {
      map.value.setLayoutProperty(layer.id, 'visibility', 'none')
    }
  })
}

// コンポーネントのマウント時の処理
onMounted(async () => {
  if (!process.client) return

  const maplibregl = await import('maplibre-gl')
  import('maplibre-gl/dist/maplibre-gl.css')

  if (mapContainer.value) {
    // 地図の初期化
    const authHelper = await withIdentityPoolId(IDENTITY_POOL_ID)
    const initialStyle = mapStyles[0]

    const mapOptions = {
      container: mapContainer.value,
      center: [139.7670, 35.6814], // 東京
      zoom: 2,
      style: initialStyle.url,
      ...authHelper.getMapAuthenticationOptions()
    }

    map.value = new maplibregl.default.Map(mapOptions)

    // スケールコントロールの追加
    map.value.addControl(
      new maplibregl.default.ScaleControl({
        maxWidth: 100,
        unit: 'metric'
      })
    )
  }
})
</script>

<style scoped>
/* コンテナのスタイル */
.map-container {
  width: 100%;
  height: 100vh;
  position: relative;
}

.map-wrapper {
  width: 100%;
  height: 100%;
}

/* スタイル切り替えパネルのスタイル */
.style-switcher {
  position: absolute;
  top: 10px;
  left: 10px;
  background: white;
  padding: 10px;
  border-radius: 4px;
  box-shadow: 0 0 10px rgba(0,0,0,0.1);
  z-index: 1;
}

/* ボタンの基本スタイル */
.style-switcher button {
  display: block;
  margin: 5px 0;
  padding: 8px 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: white;
  cursor: pointer;
  width: 100%;
}

.style-switcher button.active {
  background: #eee;
  font-weight: bold;
}

.style-switcher button:hover {
  background: #f5f5f5;
}

/* 国境線切り替えボタンのスタイル */
.border-toggle {
  margin-top: 10px;
  padding-top: 10px;
  border-top: 1px solid #ccc;
}

.border-toggle button {
  width: 100%;
  padding: 8px 16px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: white;
  cursor: pointer;
}

.border-toggle button:hover {
  background: #f5f5f5;
}

.border-toggle button.active {
  background: #eee;
  font-weight: bold;
}
</style>
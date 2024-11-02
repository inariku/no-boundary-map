<template>
  <div class="map-container">
    <div ref="mapContainer" class="map-wrapper"></div>
    <div class="style-switcher">
      <button
        v-for="style in mapStyles"
        :key="style.id"
        @click="switchStyle(style)"
        :class="{ active: currentStyle === style.id }"
      >
        {{ style.name }}
      </button>
      
      <!-- Amazon Location Service選択時のみ表示される国境線切り替えボタン -->
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
import { onMounted, ref } from 'vue'
import { Amplify } from 'aws-amplify'
import { Geo } from '@aws-amplify/geo'
import { withIdentityPoolId } from '@aws/amazon-location-utilities-auth-helper'
import outputs from '../amplify_outputs.json'

Amplify.configure(outputs)

const mapContainer = ref(null)
const map = ref(null)
const currentStyle = ref('amazon') // 初期スタイルをAmazonに変更
const bordersHidden = ref(false) // 国境線の表示状態を管理

const MAPTILER_KEY = 'ePRCzILSUpBa97QJmZwJ'
const REGION = outputs.geo.aws_region
const MAP_NAME = outputs.geo.maps.default
const IDENTITY_POOL_ID = outputs.auth.identity_pool_id

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
          tiles: [
            'https://tile.openstreetmap.org/{z}/{x}/{y}.png'
          ],
          tileSize: 256,
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }
      },
      layers: [
        {
          id: 'osm-tiles',
          type: 'raster',
          source: 'osm',
          minzoom: 0,
          maxzoom: 19
        }
      ]
    }
  },
  {
    id: 'maptiler',
    name: 'MapTiler White',
    url: `https://api.maptiler.com/maps/90b754d8-eb36-477e-bc28-337bc7bfaaf1/style.json?key=${MAPTILER_KEY}`
  }
]

// 国境線の表示/非表示を切り替える関数
const toggleBorders = () => {
  if (map.value) {
    const layers = map.value.getStyle().layers
    layers.forEach(layer => {
      if (
        layer.id.includes('border') || 
        layer.id.includes('boundary') ||
        layer.id.includes('admin')
      ) {
        map.value.setLayoutProperty(
          layer.id,
          'visibility',
          bordersHidden.value ? 'visible' : 'none'
        )
      }
    })
    bordersHidden.value = !bordersHidden.value
  }
}

const switchStyle = async (style) => {
  try {
    if (map.value) {
      const center = map.value.getCenter()
      const zoom = map.value.getZoom()
      const bearing = map.value.getBearing()
      const pitch = map.value.getPitch()

      if (style.isAmazonStyle) {
        const authHelper = await withIdentityPoolId(IDENTITY_POOL_ID)
        const newStyle = {
          style: style.url,
          ...authHelper.getMapAuthenticationOptions()
        }
        map.value.setStyle(newStyle.style, newStyle)
        
        // Amazon スタイルに切り替えた時、以前の国境線表示状態を適用
        map.value.once('style.load', () => {
          if (bordersHidden.value) {
            const layers = map.value.getStyle().layers
            layers.forEach(layer => {
              if (
                layer.id.includes('border') || 
                layer.id.includes('boundary') ||
                layer.id.includes('admin')
              ) {
                map.value.setLayoutProperty(layer.id, 'visibility', 'none')
              }
            })
          }
          map.value.setCenter(center)
          map.value.setZoom(zoom)
          map.value.setBearing(bearing)
          map.value.setPitch(pitch)
        })
      } else {
        map.value.setStyle(style.url)
        map.value.once('style.load', () => {
          map.value.setCenter(center)
          map.value.setZoom(zoom)
          map.value.setBearing(bearing)
          map.value.setPitch(pitch)
        })
      }

      currentStyle.value = style.id
    }
  } catch (error) {
    console.error('Style switch error:', error)
  }
}

onMounted(async () => {
  if (process.client) {
    const maplibregl = await import('maplibre-gl')
    import('maplibre-gl/dist/maplibre-gl.css')

    if (mapContainer.value) {
      const authHelper = await withIdentityPoolId(IDENTITY_POOL_ID)
      const initialStyle = mapStyles[0] // Amazon Location Serviceを初期スタイルに

      const mapOptions = {
        container: mapContainer.value,
        center: [139.7670, 35.6814],
        zoom: 2,
        style: initialStyle.url,
        ...authHelper.getMapAuthenticationOptions()
      }

      map.value = new maplibregl.default.Map(mapOptions)

      map.value.addControl(
        new maplibregl.default.ScaleControl({
          maxWidth: 100,
          unit: 'metric'
        })
      )
    }
  }
})
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 100vh;
  position: relative;
}

.map-wrapper {
  width: 100%;
  height: 100%;
}

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
<template>
  <div class="map-container">
    <div ref="mapContainer" class="map-wrapper"></div>
  </div>
</template>

<script setup>
import { onMounted, ref } from 'vue'

const mapContainer = ref(null)
const map = ref(null)

const MAPTILER_KEY = 'ePRCzILSUpBa97QJmZwJ'

onMounted(async () => {
  if (process.client) {
    const maplibregl = await import('maplibre-gl')
    import('maplibre-gl/dist/maplibre-gl.css')

    if (mapContainer.value) {
      map.value = new maplibregl.default.Map({
        container: mapContainer.value,
        style: `https://api.maptiler.com/maps/90b754d8-eb36-477e-bc28-337bc7bfaaf1/style.json?key=ePRCzILSUpBa97QJmZwJ`,
        center: [139.7670, 35.6814], // 東京
        zoom: 1
      })

      map.value.addControl(
        new maplibregl.default.NavigationControl()
      )
      
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
}

.map-wrapper {
  width: 100%;
  height: 100%;
}
</style>
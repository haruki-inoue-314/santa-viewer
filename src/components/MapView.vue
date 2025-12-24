<script setup lang="ts">
import { onMounted, onUnmounted, ref, watch } from 'vue'
import maplibregl from 'maplibre-gl'
import 'maplibre-gl/dist/maplibre-gl.css'
import * as turf from '@turf/turf'
import TimeSlider from './TimeSlider.vue'

interface SantaFeature {
  type: 'Feature'
  geometry: {
    type: 'Point'
    coordinates: [number, number]
  }
  properties: {
    id: string
    arrival: number
    departure: number
    population: number
    presentsDelivered: number
    city: string
    region: string
    details: {
      timezone: number
    }
  }
}

interface SantaRouteData {
  type: 'FeatureCollection'
  features: SantaFeature[]
}

const mapContainer = ref<HTMLDivElement | null>(null)
let map: maplibregl.Map | null = null
let santaMarker: maplibregl.Marker | null = null
let mapLoaded = ref(false)

const santaData = ref<SantaRouteData | null>(null)
const currentTime = ref(0)
const minTime = ref(0)
const maxTime = ref(0)

// 現在時刻に基づいてサンタの位置を計算する
const calculateSantaPosition = (timestamp: number): [number, number] | null => {
  if (!santaData.value) return null

  const features = santaData.value.features

  // 最初の地点より前の時刻
  if (timestamp <= features[0].properties.arrival) {
    return features[0].geometry.coordinates as [number, number]
  }

  // 各地点をチェック
  for (let i = 0; i < features.length - 1; i++) {
    const currentStop = features[i]
    const nextStop = features[i + 1]

    const departure = currentStop.properties.departure
    const nextArrival = nextStop.properties.arrival

    // 現在の地点に滞在中
    if (timestamp >= currentStop.properties.arrival && timestamp <= departure) {
      return currentStop.geometry.coordinates as [number, number]
    }

    // 次の地点への移動中
    if (timestamp > departure && timestamp < nextArrival) {
      const from = turf.point(currentStop.geometry.coordinates)
      const to = turf.point(nextStop.geometry.coordinates)

      // 移動時間の割合を計算
      const travelDuration = nextArrival - departure
      const elapsed = timestamp - departure
      const ratio = elapsed / travelDuration

      // 2点間の距離を計算して補間
      const distance = turf.distance(from, to, { units: 'kilometers' })
      const bearing = turf.bearing(from, to)
      const traveledDistance = distance * ratio

      const position = turf.destination(from, traveledDistance, bearing, {
        units: 'kilometers',
      })

      return position.geometry.coordinates as [number, number]
    }
  }

  // 最後の地点
  const lastFeature = features[features.length - 1]
  return lastFeature.geometry.coordinates as [number, number]
}

// 日付変更線をまたぐ座標を修正する
const fixAntimeridian = (coordinates: number[][]): number[][] => {
  if (coordinates.length <= 1) return coordinates

  const newCoords = [...coordinates.map((coord) => [...coord])] // ディープコピー

  for (let i = 1; i < newCoords.length; i++) {
    const prevLon = newCoords[i - 1][0]
    let currLon = newCoords[i][0]

    // 差分を計算
    const diff = currLon - prevLon

    // 東へ進んで日付変更線を越えた場合 (例: 170 -> -170 => 差は -340)
    // 差が -180 より小さいなら、360を足して補正
    if (diff < -180) {
      currLon += 360
    }
    // 西へ進んで日付変更線を越えた場合 (例: -170 -> 170 => 差は 340)
    // 差が 180 より大きいなら、360を引いて補正
    else if (diff > 180) {
      currLon -= 360
    }

    newCoords[i][0] = currLon
  }

  return newCoords
}

// 現在時刻までに訪れた都市を計算する
const calculateVisitedCities = (timestamp: number) => {
  if (!santaData.value) return []

  const features = santaData.value.features
  const visitedCities: SantaFeature[] = []

  for (let i = 0; i < features.length; i++) {
    const feature = features[i]

    // 到着時刻が現在時刻より前なら訪問済み
    if (feature.properties.arrival <= timestamp) {
      // 現地時刻を計算して追加
      const arrival = feature.properties.arrival
      const timezone = feature.properties.details.timezone
      // timezone は秒単位、arrival はミリ秒単位なので timezone * 1000 でミリ秒に変換
      const localTime = new Date(arrival + timezone * 1000)
      const hours = String(localTime.getUTCHours()).padStart(2, '0')
      const minutes = String(localTime.getUTCMinutes()).padStart(2, '0')

      // プロパティに現地時刻を追加
      const featureWithTime = {
        ...feature,
        properties: {
          ...feature.properties,
          localArrivalTime: `${hours}:${minutes}`,
        },
      }

      visitedCities.push(featureWithTime)
    } else {
      // まだ到着していない都市以降は追加しない
      break
    }
  }

  return visitedCities
}

// 現在時刻までの軌跡を計算する
const calculateTrailPath = (timestamp: number): number[][] => {
  if (!santaData.value) return []

  const features = santaData.value.features
  const coordinates: number[][] = []

  // 最初の地点を追加
  coordinates.push([...features[0].geometry.coordinates])

  for (let i = 0; i < features.length - 1; i++) {
    const currentStop = features[i]
    const nextStop = features[i + 1]

    const departure = currentStop.properties.departure
    const nextArrival = nextStop.properties.arrival

    // 現在時刻がこの地点の出発時刻より前なら終了
    if (timestamp < departure) {
      break
    }

    // 次の地点への移動を計算
    if (timestamp >= departure) {
      const from = turf.point(currentStop.geometry.coordinates)
      const to = turf.point(nextStop.geometry.coordinates)

      // 大圏航路を計算
      const greatCircle = turf.greatCircle(from, to, { npoints: 100 })

      // 移動完了している場合
      if (timestamp >= nextArrival) {
        // greatCircleの結果を座標配列に追加
        if (greatCircle.geometry.type === 'LineString') {
          // LineStringの場合はそのまま追加（最初の点は既に追加済みなのでスキップ）
          const gcCoords = fixAntimeridian(greatCircle.geometry.coordinates)
          for (let j = 1; j < gcCoords.length; j++) {
            coordinates.push(gcCoords[j])
          }
        } else if (greatCircle.geometry.type === 'MultiLineString') {
          // MultiLineStringの場合は結合してfixAntimeridianを適用
          const allCoords: number[][] = []
          for (const lineCoords of greatCircle.geometry.coordinates) {
            allCoords.push(...lineCoords)
          }
          const gcCoords = fixAntimeridian(allCoords)
          for (let j = 1; j < gcCoords.length; j++) {
            coordinates.push(gcCoords[j])
          }
        }
      } else {
        // 移動中の場合、補間した位置まで
        const travelDuration = nextArrival - departure
        const elapsed = timestamp - departure
        const ratio = elapsed / travelDuration

        // 大圏航路上の距離を計算
        const totalDistance = turf.distance(from, to, { units: 'kilometers' })
        const traveledDistance = totalDistance * ratio

        // greatCircleの結果をLineStringに変換（日付変更線対応）
        let lineCoords: number[][] = []
        if (greatCircle.geometry.type === 'LineString') {
          lineCoords = greatCircle.geometry.coordinates
        } else if (greatCircle.geometry.type === 'MultiLineString') {
          // MultiLineStringの場合は結合
          for (const lineSegment of greatCircle.geometry.coordinates) {
            lineCoords.push(...lineSegment)
          }
        }

        // 日付変更線をまたぐ場合の座標を修正してLineStringに変換
        const fixedCoords = fixAntimeridian(lineCoords)
        const lineString = turf.lineString(fixedCoords)

        // lineSliceAlongで進行距離に応じた部分を取得
        const sliced = turf.lineSliceAlong(lineString, 0, traveledDistance, {
          units: 'kilometers',
        })

        // スライスした座標を追加（最初の点は既に追加済みなのでスキップ）
        for (let j = 1; j < sliced.geometry.coordinates.length; j++) {
          coordinates.push([...sliced.geometry.coordinates[j]])
        }
        break
      }
    }
  }

  // 日付変更線をまたぐ座標を修正
  return fixAntimeridian(coordinates)
}

// 訪問済み都市を更新
const updateVisitedCities = () => {
  if (!map || !mapLoaded.value) return

  const visitedCities = calculateVisitedCities(currentTime.value)

  const source = map.getSource('visited-cities') as maplibregl.GeoJSONSource
  if (source) {
    source.setData({
      type: 'FeatureCollection',
      features: visitedCities,
    })
  }
}

// 軌跡を更新
const updateTrail = () => {
  if (!map || !mapLoaded.value) return

  const trail = calculateTrailPath(currentTime.value)

  const source = map.getSource('santa-trail') as maplibregl.GeoJSONSource
  if (source) {
    source.setData({
      type: 'Feature',
      properties: {},
      geometry: {
        type: 'LineString',
        coordinates: trail,
      },
    })
  }
}

// サンタの位置を更新
const updateSantaPosition = () => {
  // 訪問済み都市も更新
  updateVisitedCities()
  if (!map || !santaMarker) return

  const position = calculateSantaPosition(currentTime.value)
  if (position) {
    santaMarker.setLngLat(position)

    // 地図の中心をサンタの位置に瞬時に移動
    map.setCenter(position)
  }

  // 軌跡も更新
  updateTrail()
}

// currentTimeの変化を監視
watch(currentTime, () => {
  updateSantaPosition()
})

const loadSantaRoute = async () => {
  try {
    const response = await fetch('/santa-route.geojson')
    const data: SantaRouteData = await response.json()
    santaData.value = data

    // 時間範囲を固定値で設定
    minTime.value = 1577181600000
    maxTime.value = 1577271600000
    currentTime.value = minTime.value
  } catch (error) {
    console.error('Failed to load santa route:', error)
  }
}

onMounted(async () => {
  if (!mapContainer.value) return

  await loadSantaRoute()

  // 最初の位置を取得
  const initialPosition = santaData.value?.features[0].geometry.coordinates as [number, number]

  map = new maplibregl.Map({
    container: mapContainer.value,
    style: 'https://tile.openstreetmap.jp/styles/osm-bright-ja/style.json',
    center: initialPosition || [0, 30],
    zoom: 3,
  })

  // ナビゲーションコントロールを追加
  map.addControl(new maplibregl.NavigationControl(), 'top-right')

  // スタイルの読み込み完了を待ってGlobe Projectionを設定
  map.on('style.load', () => {
    if (!map) return
    map.setProjection({ type: 'globe' })
  })

  // 地図の読み込み完了を待つ
  map.on('load', () => {
    if (!map) return

    // 軌跡用のソースを追加
    map.addSource('santa-trail', {
      type: 'geojson',
      data: {
        type: 'Feature',
        properties: {},
        geometry: {
          type: 'LineString',
          coordinates: [],
        },
      },
    })

    // 軌跡用のレイヤーを追加
    map.addLayer({
      id: 'santa-trail-layer',
      type: 'line',
      source: 'santa-trail',
      layout: {
        'line-join': 'round',
        'line-cap': 'round',
      },
      paint: {
        'line-color': '#c41e3a',
        'line-width': 3,
        'line-opacity': 0.8,
      },
    })

    // 訪問済み都市用のソースを追加
    map.addSource('visited-cities', {
      type: 'geojson',
      data: {
        type: 'FeatureCollection',
        features: [],
      },
    })

    // 都市マーカー用のレイヤーを追加
    map.addLayer({
      id: 'city-markers',
      type: 'circle',
      source: 'visited-cities',
      paint: {
        'circle-radius': 6,
        'circle-color': '#2563eb',
        'circle-stroke-width': 2,
        'circle-stroke-color': '#ffffff',
      },
    })

    // 都市名ラベル用のレイヤーを追加
    map.addLayer({
      id: 'city-labels',
      type: 'symbol',
      source: 'visited-cities',
      layout: {
        'text-field': ['concat', ['get', 'city'], '\n', '現地時刻 ', ['get', 'localArrivalTime']],
        'text-font': ['Open Sans Regular'],
        'text-size': 11,
        'text-offset': [0, 1.8],
        'text-anchor': 'top',
      },
      paint: {
        'text-color': '#1e40af',
        'text-halo-color': '#ffffff',
        'text-halo-width': 2,
      },
    })

    mapLoaded.value = true

    // 初期位置と軌跡を設定
    updateSantaPosition()
  })

  // サンタのマーカーを作成
  const el = document.createElement('div')
  el.className = 'santa-marker'
  el.innerHTML = '🎅'
  el.style.fontSize = '32px'
  el.style.cursor = 'pointer'

  santaMarker = new maplibregl.Marker({ element: el })
    .setLngLat(initialPosition || [0, 30])
    .addTo(map)
})

onUnmounted(() => {
  santaMarker?.remove()
  santaMarker = null
  map?.remove()
  map = null
})
</script>

<template>
  <div class="map-wrapper">
    <div ref="mapContainer" class="map-container"></div>
    <TimeSlider v-if="minTime > 0" v-model="currentTime" :min-time="minTime" :max-time="maxTime" />
  </div>
</template>

<style scoped>
.map-wrapper {
  width: 100%;
  height: 100%;
}

.map-container {
  width: 100%;
  height: 100%;
}
</style>

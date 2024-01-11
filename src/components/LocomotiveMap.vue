<template>
  <div>
    <div id="map" class="fullscreen-map"></div>
  </div>
</template>

<script>
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

export default {
  //Задаём параметры leaflet
  data() {
    return {
      map: null,
      zoom: 13,
      center: [59.13262376, 37.84800256],
      url: 'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
      locoLatLong: [],
      locoTemperature: []
    }
  },
  //Загружаем карту и получаем данные
  mounted() {
    this.initMap()
    this.fetchData()
  },
  methods: {
    initMap() {
      this.map = L.map('map').setView(this.center, this.zoom)
      L.tileLayer(this.url, {
        attribution: this.attribution
      }).addTo(this.map)
    },

    async fetchData() {
      const locationData = await (await fetch('location.json')).json()
      const tempData = await (await fetch('temperatures.json')).json()

      //Проходим по массиву tempData.Timestamp, конвертируем каждый элемент в числовой формат с помощью (+new Date(timestamp)), и вычисляем абсолютное значение разницы между текущим элементом и моментом времени из locationData.Timestamp
      //Сохраняем найденную температуру в locoTemperature для каждого промежутка времени из locationData.Timestamp.
      locationData.Timestamp.forEach((timestamp) => {
        const nearestTempIndex = tempData.Timestamp.map((t) => +new Date(t)).reduce(
          (nearestI, curr, currentI, arr) =>
            Math.abs(curr - +new Date(timestamp)) < Math.abs(arr[nearestI] - +new Date(timestamp))
              ? currentI
              : nearestI,
          0
        )
        this.locoTemperature.push(tempData.OutsideTemp[nearestTempIndex])
      })
      //map() для массива путем соединения координат locationData.Latitude и locationData.Longitude. Затем используется функция filter() для удаления элементов без значений
      this.locoLatLong = locationData.Latitude.map((lat, i) => [
        lat,
        locationData.Longitude[i]
      ]).filter((latlng) => latlng[0] !== 'NA' && latlng[1] !== 'NA')

      //используем функцию L.polyline() из leaflet для создания линии с координатами из this.locoLatLong. Затем добавляем  линию на карту.
      const polyline = L.polyline(this.locoLatLong, { color: 'blue' }).addTo(this.map)
      this.map.fitBounds(polyline.getBounds())

      //Проходим по массиву this.locoLatLong и добавляем маркеры на карту.
      this.locoLatLong.forEach((latLng, i) => {
        const marker = L.circleMarker(latLng).addTo(this.map)

        //Добавляем обработчики событий на карту для каждого маркера.
        marker.on('mouseover', () => {
          marker.setStyle({ color: 'red' })
          L.popup()
            .setLatLng(latLng)
            .setContent(
              `Температура: ${this.locoTemperature[i]}°C<br>Время: ${locationData.Timestamp[i]}`
            )
            .openOn(this.map)
        })

        marker.on('mouseout', () => {
          marker.setStyle({ color: 'blue' })
          this.map.closePopup()
        })
      })
    }
  }
}
</script>

<style>
.fullscreen-map {
  position: absolute;
  top: 0;
  right: 0;
  bottom: 0;
  left: 0;
}
</style>

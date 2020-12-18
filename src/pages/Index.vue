<template>
  <q-page class="flex flex-center">
  <!--
  <img
  alt="Quasar logo"
  src="~assets/quasar-logo-full.svg"
  >
  -->
    <q-file
      v-model="stops_input"
      label="Stops"
      filled
      style="max-width: 300px"
    />
    <q-file
      v-model="stop_times_input"
      label="Stops Times"
      filled
      style="max-width: 300px"
    />
    <div>
      <q-btn label="Submit" type="submit" color="primary" @click="upload"/>
    </div>

    <div id="mapid" style="height: 100vh; width: 100%"></div>
  </q-page>
</template>

<script>
import L from 'leaflet'
import parse from 'csv-parse'

function readFileAsync (file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader()

    reader.onload = () => {
      resolve(reader.result)
    }

    reader.onerror = reject

    reader.readAsText(file)
  })
}

export default {
  name: 'PageIndex',
  data () {
    return {
      stops_input: undefined,
      stop_times_input: undefined
    }
  },
  mounted: async function () {
  },
  methods: {
    async upload () {
      var mbAttr = 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
              '<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
              'Imagery © <a href="http://mapbox.com">Mapbox</a>'
      var mbUrl = 'https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token=pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw'
      var grayscale = L.tileLayer(mbUrl, { id: 'mapbox.light', attribution: mbAttr })
      var streets = L.tileLayer(mbUrl, { id: 'mapbox.streets', attribution: mbAttr })
      var myMap = L.map('mapid', { layers: [grayscale], zoomControl: true }).setView([-33.81759, 151.005], 10)
      var baseLayers = {
        Grayscale: grayscale,
        Streets: streets
      }

      L.control.layers(baseLayers).addTo(myMap)

      if (!this.stops_input) {
        return
      }
      const stopsBuffer = await readFileAsync(this.stops_input)
      console.log(stopsBuffer)
      // var stopsReader = new FileReader()
      // stopsReader.readAsText(this.stops_input)
      const stops = {}
      // stopsReader.onloadend = async function () {
      parse(stopsBuffer, {
        comment: '#'
      }, function (err, output) {
        if (err) {
          throw err
        }
        for (let i = 1; i < output.length; i++) {
          // L.marker([output[i][3], output[i][4]]).addTo(myMap)
          stops[`${output[i][0]}`] = [output[i][3], output[i][4]]
          // L.popup()
          //   .setLatLng([output[i][3], output[i][4]])
          //   .setContent(output[i][2])
          //   .openOn(myMap)
        }
      })
      // }

      if (!this.stop_times_input) {
        return
      }
      const stopTimesBuffer = await readFileAsync(this.stop_times_input)
      // var stopTimesReader = new FileReader()
      // stopTimesReader.readAsText(this.stop_times_input)
      // stopTimesReader.onloadend = function () {
      parse(stopTimesBuffer, {
        comment: '#'
      }, function (err, output) {
        if (err) {
          throw err
        }
        let latlngs = []
        for (let i = 1; i < output.length; i++) {
          if (output[i][4] === '1') {
            latlngs = []
          } else {
            latlngs.push(stops[output[i][3].toString()])
            console.log('latlngs: ', latlngs)
            L.polyline(latlngs, { color: ['red', 'blue', 'green', 'yellow', 'purple', 'pink'][Math.floor(Math.random() * 6)] }).addTo(myMap)
          }
        }

        // zoom the map to the polyline
        // myMap.fitBounds(polyline.getBounds())
      })
      // }
    }
  }
}
</script>

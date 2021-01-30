<template>
  <q-page class="flex flex-center">
  <!--
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
  -->
    <q-btn label="Start" type="submit" color="primary" @click="start"/>
    <q-btn label="Stop" type="submit" color="primary" @click="stop"/>

    <div class="q-gutter-sm">
      <q-checkbox keep-color v-model="green" label="Green (1-10 cars)" color="green" />
      <q-checkbox keep-color v-model="yellow" label="Yellow (11-20 cars)" color="yellow" />
      <q-checkbox keep-color v-model="red" label="Red (20+ cars)" color="red" />
    </div>

    <div class="q-gutter-sm">
      <p>Green Min PT: {{this.greenPTMin}}</p>
      <p>Green Max PT: {{this.greenPTMax}}</p>
      <p>Green Mean PT: {{this.greenPTMean}}</p>
    </div>

    <div class="q-gutter-sm">
      <p>Yellow Min PT: {{this.yellowPTMin}}</p>
      <p>Yellow Max PT: {{this.yellowPTMax}}</p>
      <p>Yellow Mean PT: {{this.yellowPTMean}}</p>
    </div>

    <div class="q-gutter-sm">
      <p>Red Min PT: {{this.redPTMin}}</p>
      <p>Red Max PT: {{this.redPTMax}}</p>
      <p>Red Mean PT: {{this.redPTMean}}</p>
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
      stop_times_input: undefined,
      socket: undefined,
      myMap: undefined,
      sites: {},
      cars: {},
      green: true,
      yellow: true,
      red: true,
      greenGroup: L.featureGroup(),
      yellowGroup: L.featureGroup(),
      redGroup: L.featureGroup(),
      greenPT: [],
      yellowPT: [],
      redPT: [],
      greenPTMin: 0,
      greenPTMax: 0,
      greenPTMean: 0,
      yellowPTMin: 0,
      yellowPTMax: 0,
      yellowPTMean: 0,
      redPTMin: 0,
      redPTMax: 0,
      redPTMean: 0
    }
  },
  mounted: async function () {
    // var mbAttr = 'Map data &copy; <a href="http://openstreetmap.org">OpenStreetMap</a> contributors, ' +
    //         '<a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, ' +
    //         'Imagery © <a href="http://mapbox.com">Mapbox</a>'
    var mbUrl = 'https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}?access_token=pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw'
    // var mbUrl = 'https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token=pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw'
    var streets = L.tileLayer(mbUrl, {
      attribution: '© <a href="https://www.mapbox.com/about/maps/">Mapbox</a> © <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> <strong><a href="https://www.mapbox.com/map-feedback/" target="_blank">Improve this map</a></strong>',
      tileSize: 512,
      maxZoom: 18,
      zoomOffset: -1,
      id: 'mapbox/streets-v11',
      accessToken: 'pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw'
    })// .addTo(this.myMap)
    var grayscale = L.tileLayer(mbUrl, {
      attribution: '© <a href="https://www.mapbox.com/about/maps/">Mapbox</a> © <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a> <strong><a href="https://www.mapbox.com/map-feedback/" target="_blank">Improve this map</a></strong>',
      tileSize: 512,
      maxZoom: 18,
      zoomOffset: -1,
      id: 'mapbox/light-v10',
      accessToken: 'pk.eyJ1IjoibWFwYm94IiwiYSI6ImNpejY4NXVycTA2emYycXBndHRqcmZ3N3gifQ.rJcFIG214AriISLbB6B5aw'
    })// .addTo(this.myMap)
    // var grayscale = L.tileLayer(mbUrl, { id: 'mapbox.light', attribution: mbAttr })
    // var streets = L.tileLayer(mbUrl, { id: 'mapbox.streets', attribution: mbAttr })
    this.myMap = L.map('mapid', { layers: [grayscale], zoomControl: true }).setView([-32.0388312, 115.4010671], 10)
    var baseLayers = {
      Grayscale: grayscale,
      Streets: streets
    }

    L.control.layers(baseLayers).addTo(this.myMap)
    this.myMap.addLayer(this.greenGroup)
    this.myMap.addLayer(this.yellowGroup)
    this.myMap.addLayer(this.redGroup)

    fetch('https://mrgis.mainroads.wa.gov.au/arcgis/rest/services/OpenData/RoadAssets_DataPortal/MapServer/28/query?where=1%3D1&outFields=*&outSR=4326&f=json')
      .then(res => res.json())
      .then(
        (result) => {
          console.log(result)
          result.features.forEach(site => {
            const id = site.attributes.SITE_REFERENCE_ID
            const geometry = site.geometry
            console.log('id: ', id)
            console.log('geometry: ', geometry)
            this.sites[Number(id.substr(2))] = geometry
            // const marker = L.marker([geometry.y, geometry.x]).addTo(this.myMap)
            // marker.bindPopup(`${id}`).openPopup()
          })
        },
        // Note: it's important to handle errors here
        // instead of a catch() block so that we don't swallow
        // exceptions from actual bugs in components.
        (error) => {
          console.log('Error: ', error)
        }
      )

    fetch('https://mrgis.mainroads.wa.gov.au/arcgis/rest/services/OpenData/Traffic_DataPortal/MapServer/0/query?where=Region%20%3D%20%27METRO%27&outFields=*&outSR=4326&f=json')
      .then(res => res.json())
      .then(
        (result) => {
          console.log(result)
          result.features.forEach(incident => {
            const trafficImpact = incident.attributes.TrafficImpact
            const geometry = incident.geometry
            console.log('trafficImpact: ', trafficImpact)
            console.log('geometry: ', geometry)
            const marker = L.marker([geometry.y, geometry.x]).addTo(this.myMap)
            marker.bindPopup(`${trafficImpact}`).openPopup()
          })
        },
        // Note: it's important to handle errors here
        // instead of a catch() block so that we don't swallow
        // exceptions from actual bugs in components.
        (error) => {
          console.log('Error: ', error)
        }
      )

    // let latlngs = []
    // for (let i = 1; i < output.length; i++) {
    //   if (output[i][4] === '1') {
    //     latlngs = []
    //   } else {
    //     latlngs.push(stops[output[i][3].toString()])
    //     console.log('latlngs: ', latlngs)
    //     L.polyline(latlngs, { color: ['red', 'blue', 'green', 'yellow', 'purple', 'pink'][Math.floor(Math.random() * 6)] }).addTo(myMap)
    //   }
    // }
  },
  watch: {
    greenPT: function (val) {
      this.greenPTMin = Math.min.apply(null, val)
      this.greenPTMax = Math.max.apply(null, val)
      this.greenPTMean = Number(val.reduce((a, b) => a + b, 0) / val.length).toFixed(2)
    },
    yellowPT: function (val) {
      this.yellowPTMin = Math.min.apply(null, val)
      this.yellowPTMax = Math.max.apply(null, val)
      this.yellowPTMean = Number(val.reduce((a, b) => a + b, 0) / val.length).toFixed(2)
    },
    redPT: function (val) {
      this.redPTMin = Math.min.apply(null, val)
      this.redPTMax = Math.max.apply(null, val)
      this.redPTMean = Number(val.reduce((a, b) => a + b, 0) / val.length).toFixed(2)
    },
    green: function (val) {
      if (this.myMap.hasLayer(this.greenGroup)) {
        this.myMap.removeLayer(this.greenGroup)
      } else {
        this.myMap.addLayer(this.greenGroup)
      }
    },
    yellow: function (val) {
      if (this.myMap.hasLayer(this.yellowGroup)) {
        this.myMap.removeLayer(this.yellowGroup)
      } else {
        this.myMap.addLayer(this.yellowGroup)
      }
    },
    red: function (val) {
      if (this.myMap.hasLayer(this.redGroup)) {
        this.myMap.removeLayer(this.redGroup)
      } else {
        this.myMap.addLayer(this.redGroup)
      }
    }
  },
  methods: {
    async start () {
      // Create WebSocket connection.
      this.socket = new WebSocket('wss://mainroadsopendata.mainroads.wa.gov.au:8182')

      // Connection opened
      this.socket.addEventListener('open', function (event) {
        this.socket.send('Hello Server!')
      })

      // Listen for messages
      this.socket.addEventListener('message', function (event) {
        const data = JSON.parse(event.data)
        data.forEach(message => {
          const cycleTime = message.CT
          console.log('cycleTime: ', cycleTime)
          const measures = message.SM
          measures.forEach(measure => {
            const phaseTime = measure.PT
            console.log('phaseTime: ', phaseTime)
            const id = measure.LM
            console.log('id: ', id)
            const lanes = measure.Lanes
            lanes.forEach(lane => {
              const laneNumber = lane.L
              const cars = lane.MF
              console.log('lane: ', laneNumber)
              console.log('cars: ', cars)
              this.sites[id].cars = cars
              const site = this.sites[id]
              console.log('site: ', site)
              let color = ''
              let circle
              switch (true) {
                case (cars >= 1 && cars < 10):
                  color = 'green'
                  circle = L.circle([site.y, site.x], {
                    color: color,
                    fillOpacity: 0.1,
                    radius: 300
                  }).addTo(this.greenGroup)
                  this.greenPT.push(phaseTime)
                  break
                case (cars >= 10 && cars < 20):
                  color = 'yellow'
                  circle = L.circle([site.y, site.x], {
                    color: color,
                    fillOpacity: 0.1,
                    radius: 300
                  }).addTo(this.yellowGroup)
                  this.yellowPT.push(phaseTime)
                  break
                case (cars >= 20):
                  color = 'red'
                  circle = L.circle([site.y, site.x], {
                    color: color,
                    fillOpacity: 0.1,
                    radius: 300
                  }).addTo(this.redGroup)
                  this.redPT.push(phaseTime)
                  break
                default:
                  break
              }
              if (color === '') return
              circle.bindTooltip(`lane ${laneNumber} has ${cars} cars <br/> in ${phaseTime} seconds`)
            })
          })
        })
      }.bind(this))
    },
    async stop () {
      this.socket.close()
      console.log('Closing')
    },
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

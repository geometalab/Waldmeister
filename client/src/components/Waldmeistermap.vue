<template>
  <div id="app">
    <v-layout row justify-center>
      <v-dialog v-model="save_dialog" max-width="500px">
        <v-card>
          <v-card-title>
            <span>Create new Userarea</span>
            <v-spacer></v-spacer>
          </v-card-title>
          <v-card-text>
            <v-text-field label="Label" type="text" v-model="userAreaLabel"></v-text-field>
            Public Area:
            <input type="checkbox" id="checkbox" v-model="checked">
            <label for="checkbox"> {{ checked }}</label>
          </v-card-text>
          <v-card-actions>
            <v-btn color="primary" flat @click.stop="save_dialog=false">Close</v-btn>
            <v-btn color="primary" dark class="light-green" flat @click="save">Save</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-layout>
    <div id='map'>
    </div>
  </div>
</template>
<script>
import AreaService from '@/services/AreaService'
import Vegetation from '@/components/data/vegetationskarte_minimal.json'

var myGeoJsonPoly = [];
var myCoords
var PolyCoordinates

export default {
  data() {
    return {
      checked: "",
      userAreaLabel: "",
      vegetation: Vegetation,
      title: 'WaldmeisterMap',
      zoom: 13,
      center: [51.505, -0.09],
      url: "http://{s}.tile.osm.org/{z}/{x}/{y}.png",

      save_dialog: false,
      notifications: false,
      sound: true,
      widgets: false
    }
  },
  methods: {
    save() {
      //console.log(myGeoJsonPoly)
      var theArea = {
        "label": this.userAreaLabel,
        "public": this.checked,
        "polygon": {
          "type": "MultiPolygon",
          "coordinates": [
            [myGeoJsonPoly]
          ]
        }
      }
      AreaService.postArea(theArea);
      this.save_dialog = false;
      myGeoJsonPoly = [];
    }
  },

  async mounted() {
    console.log("Loading Vegetationmap")
    //console.log(this.vegetation);
    var startPoint = [47.4348826, 8.7460494];
    var map = L.map('map', { editable: true }).setView(startPoint, 15),
      tilelayer = L.tileLayer('https://stamen-tiles-{s}.a.ssl.fastly.net/terrain/{z}/{x}/{y}.{ext}', {
        maxZoom: 20,
        attribution: 'Map tiles by <a href="http://stamen.com">Stamen Design</a>, <a href="http://creativecommons.org/licenses/by/3.0">CC BY 3.0</a> &mdash; Map data &copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
        subdomains: 'abcd',
        minZoom: 0,
        maxZoom: 18,
        ext: 'png'
      }).addTo(map);
      //Geolocation and Marker
      map.locate({setView: true, maxZoom: 15, enableHighAccuracy:false, timeout:60000, maximumAge:Infinity});
      

    L.NewPolygonControl = L.Control.extend({

      options: {
        position: 'topleft'
      },

      onAdd: function(map) {
        var container = L.DomUtil.create('div', 'leaflet-control leaflet-bar'),
          link = L.DomUtil.create('a', '', container);

        link.href = '#';
        link.title = 'Create a new polygon';
        link.innerHTML = 'Add';
        L.DomEvent.on(link, 'click', L.DomEvent.stop)
          .on(link, 'click', function() {
            map.editTools.startPolygon();
          });
        container.style.display = 'block';
        map.editTools.on('editable:enabled', function(e) {
          container.style.display = 'none';
        });
        map.editTools.on('editable:disable', function(e) {
          container.style.display = 'block';
        });
        return container;
      }
    });


    var UserAreaGroup = L.layerGroup();

    L.AddPolygonShapeControl = L.Control.extend({

      options: {
        position: 'topleft'
      },

      onAdd: function(map) {
        var container = L.DomUtil.create('div', 'leaflet-control leaflet-bar'),
          link = L.DomUtil.create('a', '', container);

        link.href = '#';
        link.title = 'Create a new polygon';
        link.innerHTML = 'Edit';
        L.DomEvent.on(link, 'click', L.DomEvent.stop)
          .on(link, 'click', function() {
            if (!map.editTools.currentPolygon) return;
            map.editTools.currentPolygon.editor.newShape();
          });
        container.style.display = 'none';
        map.editTools.on('editable:enabled', function(e) {
          container.style.display = 'block';
        });
        map.editTools.on('editable:disable', function(e) {
          container.style.display = 'none';
        });

        return container;
      }
    });
    var self = this;

    L.GeoJsonControl = L.Control.extend({

      options: {
        position: 'topleft'
      },

      onAdd: function(map) {
        var container = L.DomUtil.create('div', 'leaflet-control leaflet-bar'),
          link = L.DomUtil.create('a', '', container);

        link.href = '#';
        link.title = 'Toggle Vegetation Layer';
        link.innerHTML = 'Veg';
        L.DomEvent.on(link, 'click', L.DomEvent.stop)
          .on(link, 'click', function() {
            //map.editTools.startPolygon();
            if (self.$store.state.toggleVegetation) {
              myGeoJsonLayer.clearLayers();
              labelGroup.clearLayers();
              self.$store.dispatch('toggleVegetation', null)
            } else { 
              myGeoJsonLayer.addData(self.vegetation);
              self.$store.dispatch('toggleVegetation', null)
            }

          });
        container.style.display = 'block';
        return container;
      }
    });

    L.UserAreasControl = L.Control.extend({

      options: {
        position: 'topleft'
      },

      onAdd: function(map) {
        var container = L.DomUtil.create('div', 'leaflet-control leaflet-bar'),
          link = L.DomUtil.create('a', '', container);

        link.href = '#';
        link.title = 'Toggle UserAreas Layer';
        link.innerHTML = 'Areas';
        L.DomEvent.on(link, 'click', L.DomEvent.stop)
          .on(link, 'click', function() {
            //map.editTools.startPolygon();
            if (self.$store.state.toggleUserAreas) {
              UserAreaGroup.clearLayers();
              self.$store.dispatch('toggleUserAreas', null)
            } else { 
              DrawAllUserAreas()
              self.$store.dispatch('toggleUserAreas', null)
            }

          });
        container.style.display = 'block';
        return container;
      }
    });

    map.addControl(new L.NewPolygonControl());
    map.addControl(new L.AddPolygonShapeControl());
    map.addControl(new L.GeoJsonControl());
    map.addControl(new L.UserAreasControl());



    map.on('layeradd', function(e) {
      if (e.layer instanceof L.Polygon) e.layer.on('click', L.DomEvent.stop).on('click', e.layer.toggleEdit);
    });

    map.on('layerremove', function(e) {
      if (e.layer instanceof L.Polygon) e.layer.off('click', L.DomEvent.stop).off('click', e.layer.toggleEdit);
    });


    map.editTools.on('editable:enable', function(e) {
      if (this.currentPolygon) this.currentPolygon.disableEdit();
      this.currentPolygon = e.layer;
      this.fire('editable:enabled');
    });

    map.editTools.on('editable:disable', function(e) {
      delete this.currentPolygon;
    });


    var self = this;

    map.editTools.on('editable:drawing:commit', function(e) {
      console.log("stoppedediting");

      self.save_dialog = true;

      //console.log(this.currentPolygon);
      //console.log(this.currentPolygon.editor.tools.currentPolygon._latlngs);
      var type = e.layerType,
        layer = e.layer;
      var point = layer.getLatLngs();

      //console.log(myGeoJsonPoly.geometry.coordinates);
      console.log("before")
      //console.log(myGeoJsonPoly.geometry.coordinates);
      console.log(point);
      //myGeoJsonPoly[0][myGeoJsonPoly[0].length] = myGeoJsonPoly[0][0]
      //console.log("after");
      //console.log(myGeoJsonPoly);


      for (var i = 0; i < point[0].length; i++) {
        myGeoJsonPoly.push([
          point[0][i].lat,
          point[0][i].lng
        ]);
        console.log("i " + i);
        console.log(point[0][i]);
        console.log(myGeoJsonPoly[i]);
      }

      myGeoJsonPoly
        .push([
          point[0][0].lat,
          point[0][0].lng
        ]);
    })

    //Show my location on map
    var options = {
      enableHighAccuracy: false,
      timeout: 60000,
      maximumAge: Infinity
    };

    function success(pos) {
      var crd = pos.coords;

      console.log('Your current position is:');
      console.log(`Latitude : ${crd.latitude}`);
      console.log(`Longitude: ${crd.longitude}`);
      console.log(`More or less ${crd.accuracy} meters.`);
      L.circle([crd.latitude, crd.longitude], crd.accuracy).addTo(map);
      console.log("Added geolocation circle");
    }

    function error(err) {
      console.warn(`ERROR(${err.code}): ${err.message}`);
    }

    navigator.geolocation.getCurrentPosition(success, error, options);

    //Add geojson Layer and styling
    //var myGeoJsonLayer = L.geoJSON(this.vegetation, {
    var labelGroup = L.layerGroup();
    var myGeoJsonLayer = L.geoJSON(undefined, {
      style: function(feature){
        switch (feature.properties.EK72) {
          case '1':
            return { color: "#f2142b" };
          case '2':
            return { color: "#f20a50" };
          case '6':
            return { color: "#de3629" };
          case '7a':
            return { color: "#a45930" };
          case '7as':
            return { color: "#008e83" };
          case '7b':
            return { color: "#cb006c" };
          case '7e':
            return { color: "#c83025" };
          case '7f':
            return { color: "#6c6832" };
          case '7g':
            return { color: "#00a772" };
          case '7stern':
            return { color: "#cb006c" };
          case '8a':
            return { color: "#8c2e22" };
          case '8as':
            return { color: "#008e84" };
          case '8d':
            return { color: "#c11524" };
          case '8e':
            return { color: "#744531" };
          case '8f':
            return { color: "#644633" };
          case '8g':
            return { color: "#00a773" };
          case '8stern':
            return { color: "#c9057d" };
          case '9':
            return { color: "#9ebd4b" };
          case '10':
            return { color: "#bdc156" };
          case '10w':
            return { color: "#ece650" };
          case '11':
            return { color: "#008c4e" };

          default:
          return { color: "005800" };
        }

      },
      "weight": 0.8,
      "opacity": 0.75,
      "editable": false,

      //Draws labels for the Polygons
      onEachFeature: function(feature, layer) {
        var label = L.marker(layer.getBounds().getCenter(), {
          icon: L.divIcon({
            className: 'propertyLabel',
            html: feature.properties.EK72,
            iconSize: [20, 20],
            direction: 'auto'
          })
        }).addTo(labelGroup);
      }
      
    }).addTo(map);
    labelGroup.addTo(map);


    if (self.$store.state.toggleVegetation) {
      myGeoJsonLayer.addData(self.vegetation);
    }
    
    DrawAllUserAreas();
    //DRAWS ALL POLYGONS
    async function DrawAllUserAreas(){
      self.MyAreas = (await AreaService.getAreas()).data

      

      var val;
      for (val of self.MyAreas) {
        //console.log(val.polygon.coordinates[0][0]);
        var corner;
        var polygonToAdd = [];
        for (corner of val.polygon.coordinates[0][0]) {
          //console.log("corner found " + val.label + corner);
          polygonToAdd.push(corner);
        }
        //console.log(polygonToAdd);
        var poly = L.polygon([
          [
            polygonToAdd
          ]
        ],

        ).addTo(UserAreaGroup);

        if (val.public == true) {
          var label = L.marker(poly.getBounds().getCenter(), {
            icon: L.divIcon({
              className: 'AreaLabelPublic',
              html: val.label,
              iconSize: [100, 0],
              direction: 'auto',
            })
          }).addTo(UserAreaGroup);
        } else {
          var label = L.marker(poly.getBounds().getCenter(), {
            icon: L.divIcon({
              className: 'AreaLabelPrivate',
              html: val.label,
              iconSize: [100, 0],
              direction: 'auto'
            })
          }).addTo(UserAreaGroup);
        }
        UserAreaGroup.addTo(map);
        //poly.enableEdit();

      }
    };

    

            
    //MAP LEGEND
    var legend = L.control({ position: 'topright' });

    legend.onAdd = function(map) {

      var div = L.DomUtil.create('div', 'info legend'),
        grades = [0, 10, 20, 50, 100, 200, 500, 1000],
        labels = [];

      return div;
    };

    legend.addTo(map);
    //console.log(this.MyAreas[1].polygon.coordinates[0][0][0])

  }

}

</script>
<style type='text/css'>
body {
  margin: 0;
  padding: 0;
}

#map {
  position: absolute;
  top: 0;
  bottom: 0;
  right: 0;
  left: 0;
  width: 100%;
}

.leaflet-sidebar>.leaflet-control {
  overflow-y: hidden;
}

.propertyLabel {
  color: white;
  text-shadow: 2px 2px 2px black;
  opacity: 1;
}

.cssPolygon{
  color:green;
}
.AreaLabelPublic {
  color: green;
  text-shadow: 2px 2px 2px black;
  opacity: 1;
}

.AreaLabelPrivate {
  color: yellow;
  text-shadow: 2px 2px 2px black;
  opacity: 1;
}

div.overlay--active {
  z-index: 1000 !important
}

div.dialog__content__active {
  z-index: 1000 !important
}

</style>
<!-- <style lang="scss">
@import './assets/leaflet.css';
</style> -->

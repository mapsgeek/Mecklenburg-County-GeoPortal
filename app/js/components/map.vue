<template lang="html">
  <div class="map-container">
      <div id="map"></div>
      <button id="map-menu" class="mdl-button mdl-js-button mdl-button--icon">
          <i class="material-icons">more_vert</i>
      </button>
      <ul class="mdl-menu mdl-menu--bottom-right mdl-js-menu mdl-js-ripple-effect map-menu-list" for="map-menu">
          <li class="mdl-menu__item map-fullscreen" v-show="privateState.fullScreen" v-on:click="mapFullscreen()">Toggle Full Screen</li>
          <li class="mdl-menu__item" v-on:click="mapSatellite()">Toggle Satellite</li>
          <li class="mdl-menu__item" v-on:click="mapPitch()">Toggle Pitch</li>
      </ul>
  </div>
</template>

<script>
import Axios from 'axios';
//import mapboxgl from 'mapbox-gl';
import mapboxgl from 'mapbox-gl/src/index.js';
import directions from '../modules/directions';
import fetchNearest from '../modules/nearest';


export default {
    name: 'map',
    mounted: function() {
        this.initMap();

        // check for full screen support
        let elem = document.querySelector('.map-container');
        if (!elem.requestFullscreen && !elem.mozRequestFullScreen && !elem.webkitRequestFullScreen && !elem.msRequestFullscreen) {
            this.privateState.fullScreen = false;
        }
    },
    watch: {
        "sharedState.selected.lnglat": "addMarker",
        "sharedState.mapOverlay": "mapOverlay",
        "sharedState.poi": "addPOI"
    },
    methods: {
        initMap: function() {
            let _this = this;

            let mapOptions = {
                container: 'map',                
                style: './style/osm-mecklenburg.json',
                attributionControl: false,
                center: [-80.84, 35.26],
                zoom: 8.5,
                minZoom: 8,
                maxBounds: [[-83.285, 33.180],[-78.255, 37.384]]
            };

            _this.privateState.map = new mapboxgl.Map(mapOptions);
            let map = _this.privateState.map;

            map.on('load', function() { 
                _this.mapOverlay();
            });

            map.on('click', function (e) {
                if (map.getZoom() >= 14 && !_this.privateState.markerClicked) {
                     fetchNearest(e.lngLat.lat, e.lngLat.lng, _this.sharedState);
                }
                _this.privateState.markerClicked = false;
            });

        },
        mapPitch: function() {
            this.privateState.map.getPitch() === 0 ? this.privateState.map.easeTo({pitch: 90}) : this.privateState.map.easeTo({pitch: 0, bearing: 0});
        },
        mapOverlay: function() {
            let map = this.privateState.map;
            let _this = this;

            if (map.getLayer("overlay")) {
                map.removeLayer("overlay");
                map.removeSource("overlay");
            }

            if (_this.sharedState.mapOverlay && _this.sharedState.mapOverlay === 'view_regulated_floodplains') {                
                map.addLayer({
                    "id": "overlay",
                    "type": "fill",
                    "source": {
                    "type": "vector",
                        "tiles": [
                            "https://mcmap.org/api/mvt/v1/view_regulated_floodplains/{z}/{x}/{y}?geom_column=the_geom"
                        ],
                        "maxzoom": 14,
                        "minzoom": 14
                    },
                    "source-layer": "view_regulated_floodplains",
                    "minzoom": 14,
                    "maxzoom": 22,
                    "paint": {
                        "fill-color": "#DDF7F7",
                    }
                }, 'waterway_river');
            }

            if (_this.sharedState.mapOverlay && _this.sharedState.mapOverlay === 'impervious_surface') {                
                map.addLayer({
                    "id": "overlay",
                    "type": "raster",
                    "source": {
                        "type": "raster",
                        "tiles": [`https://mcmap.org/geoserver/postgis/wms?bbox={bbox-epsg-3857}&format=image/png&service=WMS&version=1.1.1&request=GetMap&srs=EPSG:3857&width=256&height=256&layers=postgis:impervious_surface&transparent=true`],
                        "tileSize": 256,
                        "maxzoom": 18
                    },
                    "minzoom": 15,
                    "maxzoom": 22,
                    "paint": {
                        "raster-opacity": 1
                    }
                }, 'water_label');
                
                // map.addLayer({
                //     "id": "overlay",
                //     "type": "fill",
                //     "source": {
                //     "type": "vector",
                //         "tiles": [
                //             "https://mcmap.org/api/mvt/v1/impervious_surface/{z}/{x}/{y}?columns=type"
                //         ],
                //         "maxzoom": 14,
                //         "minzoom": 14
                //     },
                //     "source-layer": "impervious_surface",
                //     "minzoom": 14,
                //     "maxzoom": 22,
                //     "paint": {
                //         "fill-color": {
                //             property: 'type',
                //             type: 'categorical',
                //             stops: [
                //                 ['Residential', '#CBD9A5'],
                //                 ['Commercial', '#DAB2C4']
                //             ]
                //         }
                //     }
                // }, 'water_label');
            }
        },
        mapSatellite: function() {
            let map = this.privateState.map;
            if (map.getLayer("satellite")) {
                map.removeLayer("satellite");
            } else {
                map.addLayer({
                    "id": "satellite",
                    "type": "raster",
                    "source": "satellite",
                    "minzoom": 0,
            		"maxzoom": 22
                }, 'water_label');
            }
        },
        mapFullscreen: function() {
            let elem = document.querySelector(".map-container");
            if (!document.fullscreenElement && !document.mozFullScreenElement && !document.webkitFullscreenElement && !document.msFullscreenElement ) {  // current working methods
                if (elem.requestFullscreen) {
                    elem.requestFullscreen();
                } else if (elem.msRequestFullscreen) {
                    elem.msRequestFullscreen();
                } else if (elem.mozRequestFullScreen) {
                    elem.mozRequestFullScreen();
                } else if (elem.webkitRequestFullscreen) {
                    elem.webkitRequestFullscreen(Element.ALLOW_KEYBOARD_INPUT);
                }
            } else {
                if (document.exitFullscreen) {
                    document.exitFullscreen();
                } else if (document.msExitFullscreen) {
                    document.msExitFullscreen();
                } else if (document.mozCancelFullScreen) {
                    document.mozCancelFullScreen();
                } else if (document.webkitExitFullscreen) {
                    document.webkitExitFullscreen();
                }
            }
        },
        addPOI: function() {
            let map = this.privateState.map;
            let selected = this.sharedState.selected;
            let poi = this.sharedState.poi;
            let _this = this;

            // zoom to selected and poi
            let bounds = new mapboxgl.LngLatBounds();
            bounds.extend(poi.lnglat);
            bounds.extend(selected.lnglat);
            map.fitBounds(bounds, {padding: 40});

            // remove old markers if they exist
            if (this.privateState.poiMarker) {
                this.privateState.poiMarker.remove()
            }

            // create the popup
            let popup = new mapboxgl.Popup({offset:[2, -23]})
                .setHTML(`<strong>${poi.label}</strong><br>${poi.address}<br><a href="${directions(selected.lnglat, poi.lnglat)}" target="_blank" title="directions">Directions</a>`);

            // create DOM element for the marker
            var el = document.createElement('div');
            el.classList.add('poiMarker');

            // create the marker
            this.privateState.poiMarker = new mapboxgl.Marker(el, {offset:[-20, -20]})
                .setLngLat(poi.lnglat)
                .setPopup(popup) // sets a popup on this marker
                .addTo(map);

            document.querySelector('.poiMarker').addEventListener('click', function(e) {
                _this.privateState.markerClicked = true;
            });
        },
        addMarker: function() {
            let map = this.privateState.map;
            let selected = this.sharedState.selected;
            let _this = this;

            map.flyTo({
                center: selected.lnglat,
                zoom: 17
            });

            // push state
            history.replaceState(null, null, `?q=${_this.sharedState.show}&latlng=${selected.lnglat[1]},${selected.lnglat[0]}`);

            // create the popup
            let popup = new mapboxgl.Popup({offset:[2, -23]})
                .setHTML(`<strong>${selected.label}</strong><br>${selected.address}`);

            // create DOM element for the marker
            var el = document.createElement('div');
            el.classList.add('locationMarker');

            // remove old markers if they exist
            if (this.privateState.locationMarker) {
                this.privateState.locationMarker.remove()
            }
            if (this.privateState.poiMarker) {
                this.privateState.poiMarker.remove()
            }

            // create the marker
            this.privateState.locationMarker = new mapboxgl.Marker(el, {offset:[-20, -20]})
                .setLngLat(selected.lnglat)
                .setPopup(popup) // sets a popup on this marker
                .addTo(map);

            document.querySelector('.locationMarker').addEventListener('click', function(e) {
                _this.privateState.markerClicked = true;
            });
        }
    }
}
</script>

<style lang="css">
.locationMarker {
    background-image: url('./img/location.svg');
    background-size: cover;
    width: 40px;
    height: 40px;
    cursor: pointer;
}
.poiMarker {
    background-image: url('./img/poi.svg');
    background-size: cover;
    width: 40px;
    height: 40px;
    cursor: pointer;
}
</style>

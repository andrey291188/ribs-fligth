<template>
  <div>
    <div ref="map" id="maps"></div>
    <button @click="toggleMovement">{{ movementButtonLabel }}</button>
  </div>
</template>

<script>
import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import { Tile as TileLayer } from 'ol/layer';
import { OSM, XYZ } from 'ol/source';
import { Feature } from 'ol';
import { Point } from 'ol/geom';
import { Vector as VectorLayer } from 'ol/layer';
import { Vector as VectorSource } from 'ol/source';
import { Style, Icon } from 'ol/style';
import data from "../service/data.json";

export default {
  name: 'OpenLayersMap',
  data() {
    return {
      marker: null,
      map: null,
      latitude: 0,
      longitude: 0,
      isMovementRunning: false,
      timeoutId: null,
    };
  },
  computed: {
    movementButtonLabel() {
      return this.isMovementRunning ? 'Стоп' : 'Старт';
    },
  },
  mounted() {
    this.map = new Map({
      target: this.$refs.map,
      layers: [
        new TileLayer({
          source: new OSM(),
        }),
        new TileLayer({
          source: new XYZ({
            url: 'http://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
          }),
        }),
      ],
      view: new View({
        center: [0, 0],
        zoom: 2,
      }),
    });

    this.marker = new Feature({
      geometry: new Point([0, 0]),
    });

    const markerStyle = new Style({
      image: new Icon({
        src: 'https://openlayers.org/en/latest/examples/data/icon.png',
        anchor: [0.5, 1],
        scale: 1,
      }),
    });

    this.marker.setStyle(markerStyle);

    const vectorLayer = new VectorLayer({
      source: new VectorSource({
        features: [this.marker],
      }),
    });
    this.map.addLayer(vectorLayer);
  },
  methods: {
    updateMarkerCoordinates(data) {
      let index = 0;
      const delay = 200;

      const updateNextMarker = () => {
        if (index < data.length) {
          const { latitude, longitude } = data[index];
          this.marker.getGeometry().setCoordinates([longitude, latitude]);
          this.map.getView().setCenter(this.marker.getGeometry().getCoordinates())
          index += 1;
          this.timeoutId = setTimeout(updateNextMarker, delay);
        }
      };

      updateNextMarker();
    },
    calculateNewCoordinates(speed, direction, timestamp) {
      const speedInKmPerSecond = parseFloat(speed) / (60 * 60 * 1000);
      const directionInRadians = parseFloat(direction) * Math.PI / 180;

      const distanceTraveled = speedInKmPerSecond * timestamp;

      const distanceLatitude = distanceTraveled * Math.cos(directionInRadians); // Расстояние по широте
      const distanceLongitude = distanceTraveled * Math.sin(directionInRadians)

      const newLatitude = this.latitude + distanceLatitude;
      const newLongitude = this.longitude + distanceLongitude;

      this.latitude = newLatitude;
      this.longitude = newLongitude;

      return { latitude: newLatitude, longitude: newLongitude };
    },
    toggleMovement() {
      if (this.isMovementRunning) {
        this.stopMovement();
      } else {
        this.startMovement();
      }
    },
    startMovement() {
      this.updateMarkerCoordinates(data.map(({ speed, direction, timestamp }) => this.calculateNewCoordinates(speed, direction, timestamp)));
      this.isMovementRunning = true;
    },
    stopMovement() {
      this.latitude = 0,
        this.longitude = 0,
        this.marker.getGeometry().setCoordinates([this.longitude, this.latitude]);
      this.map.getView().setCenter(this.marker.getGeometry().getCoordinates())
      clearTimeout(this.timeoutId)
      this.timeoutId = null;
      this.isMovementRunning = false;

    },
  },
};
</script>

<style scoped>
#maps {
  display: flex;
  justify-content: center;
  align-items: center;

  width: 800px;
  height: 500px;
}
</style>
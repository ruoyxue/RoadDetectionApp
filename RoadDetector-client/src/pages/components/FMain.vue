<template>
	<div id="map-viewer"></div>
	<div id="geocoder-box"></div>

    <el-progress type="circle" :percentage="$store.state.detectionInfo.progress" 
            :stroke-width="14" :width="160" :color="progressBarColors" id="progress-bar">
        <span id="percentage-label">{{ $store.state.detectionInfo.step }}</span>
    </el-progress>
</template>

<script setup>
import mapboxgl from 'mapbox-gl';
import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder';
import MapboxLanguage from '@mapbox/mapbox-gl-language';
import MapboxDraw from "@mapbox/mapbox-gl-draw";
import DrawRectangle from '~/composables/DrawRectangle'
import 'mapbox-gl/dist/mapbox-gl.css';
import '@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.css';
import "@mapbox/mapbox-gl-draw/dist/mapbox-gl-draw.css";
import { onMounted, onBeforeUnmount, ref, watch, computed } from 'vue';
import { useStore } from 'vuex';

const store = useStore()

let latlon = ref('')

let map, geocoder, draw

onBeforeUnmount(() => {
    map = null
});

onMounted(() => {
    init();
});


function init() {
	// create map
    mapboxgl.accessToken = 'pk.eyJ1IjoieHVlcnVveWFvIiwiYSI6ImNsYW05cjZ0dDA2dWEzdmxzajV3bmN4Z2YifQ.g_zwCEixkEiiiVaL20TsHQ'
    map = new mapboxgl.Map({
        container: 'map-viewer',
        style: 'mapbox://styles/mapbox/satellite-streets-v12',
        center: [102.3, 35.66],
        zoom: 2,
        projection: 'globe', // 为 3D 地球
        antialias: false,
		maxZoom: 24,
		minZoom: 0,
    });
    window.matchMedia('(prefers-reduced-motion: no-preference)');
	
    // add geocoder
	geocoder = new MapboxGeocoder({ 
		accessToken: mapboxgl.accessToken,
		localGeocoder: coordinatesGeocoder,
		language: 'en',
        placeholder: 'Location or Lng,Lat',
		flyTo: {
			bearing: 1,
			speed: 1, 
			curve: 1, 
		},
		mapboxgl: mapboxgl
	})
	document.getElementById('geocoder-box').appendChild(geocoder.onAdd(map));

	// add draw tools
	let modes = MapboxDraw.modes;
	modes.draw_rectangle = DrawRectangle;
	draw = new MapboxDraw({
		modes: modes,
		displayControlsDefault: false,
	});
	map.addControl(draw);
	map.on('draw.create', function (e) {});

	// language
	// mapboxgl.setRTLTextPlugin('https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-rtl-text/v0.2.3/mapbox-gl-rtl-text.js');
	map.addControl(new MapboxLanguage({ defaultLanguage: 'en' }));

    // navigation control
    map.addControl(new mapboxgl.NavigationControl({
            visualizePitch: true,
            showCompass: true,
            showZoom: false
        }), 'top-right')

	store.commit('SET_Map', map)
	store.commit('SET_Draw', draw)
}

const coordinatesGeocoder = function (query) {
	// Match anything which looks like
	// decimal degrees coordinate pair.
	const matches = query.match(/^[ ]*(?:Lat: )?(-?\d+\.?\d*)[, ]+(?:Lng: )?(-?\d+\.?\d*)[ ]*$/i)
	if (!matches) {
		return null
	}
	
	function coordinateFeature(lng, lat) {
		return {
			center: [lng, lat],
			geometry: {
				type: 'Point',
				coordinates: [lng, lat]
			},
			place_name: 'Lat: ' + lat + ' Lng: ' + lng,
			place_type: ['coordinate'],
			properties: {},
			type: 'Feature'
		}
	}
	
	const coord1 = Number(matches[1])
	const coord2 = Number(matches[2])
	const geocodes = []
	
	if (coord1 < -90 || coord1 > 90) {
		// must be lng, lat
		geocodes.push(coordinateFeature(coord1, coord2))
	}
	
	if (coord2 < -90 || coord2 > 90) {
		// must be lat, lng
		geocodes.push(coordinateFeature(coord2, coord1))
	}
	
	if (geocodes.length === 0) {
		// else could be either lng, lat or lat, lng
		geocodes.push(coordinateFeature(coord1, coord2))
		geocodes.push(coordinateFeature(coord2, coord1))
	}
	
	return geocodes;
};

// progress bar
const progressBarColors = [
    { color: '#f56c6c', percentage: 20 },
    { color: '#e6a23c', percentage: 40 },
    { color: '#5cb87a', percentage: 60 },
    { color: '#6f7ad3', percentage: 80 },
    { color: '#1989fa', percentage: 100 },
]

</script>


<style scoped>
#map-viewer { 
    top: 48px;
	bottom: 0px;
	left: 0px;
	right: 0px;
	z-index:-5;
	@apply fixed;
}

#geocoder-box {
	top: 100px;
	left: 40px;
	width: 350px;
	z-index: 100;
	pointer-events: none;
	@apply absolute flex justify-start;
}

#progress-bar {
    left: 40px;
    bottom: 3%;
    position: absolute;
    pointer-events: none;
}

#percentage-label {
    font-weight: 900;
    @apply text-light-900 text-2xl;
}

#latlon-info {
	position: absolute;
	height: 70px;
	width: 200px;
	left: 40px;
	bottom: 35px;
	pointer-events: none;
	z-index: 100;
	@apply mt-auto text-stroke-sm text-light-500 bg-neutral-600 border-0;
}

:deep(.mapboxgl-ctrl-group) {
    @apply mr-5 mt-3;
}

:deep(.mapboxgl-ctrl-compass) {
    height: 35px;
    width: 35px;
}

:deep(.mapboxgl-ctrl-scale) {
    height: 30px;
    font-size: 17px;
    font-weight: bold;
    border: 3px solid;
    border-top: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    @apply select-none;
}

:deep(.el-card__header) {
	@apply pt-2 pb-0 flex justify-center items-center border-0;
}

:deep(.el-card__body) {
	@apply pt-1.5 pb-1 px-0 flex justify-center items-center border-0;
}

:deep(.mapboxgl-ctrl-geocoder) {
	min-width: 100%;
}

:deep(.mapboxgl-ctrl-logo) {
    display: none;
}

</style>
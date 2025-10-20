<!-- eslint-disable @typescript-eslint/no-explicit-any -->

<script setup lang="ts">
import { AttributionControl, Map, NavigationControl } from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css'
import '@maplibre/maplibre-gl-compare/dist/maplibre-gl-compare.css'
import { computed, onMounted, ref, watch } from 'vue';
import { isMapboxURL, transformMapboxUrl } from './libs/mapbox-transform';
import * as d3 from "d3";

import cunygclogo from './cunygc_logo.png'

import resultsRaw from './stores/results.geojson?raw';
import InfoBox from './InfoBox.vue'
import Legend from './Legend.vue';

// consts
const MAPBOX_KEY = 'pk.eyJ1IjoiY3VueWN1ciIsImEiOiJfQmNSMF9NIn0.uRgbcFeJbw2xyTUZY8gYeA'
const MAPBOX_STYLE_URL = 'mapbox://styles/cunycur/cm2yzn8mp00ox01pa3oxc4syx';
const COLOR_SCALE = {
    'breakpoints': [
        5, 10, 25, 50, 75, 100
    ],
    'candidates': [
        {
            // id must match the winerColumn (winner25) value and vote share column name + 25 (mamdani25)
            'id': 'Mamdani',
            // label for legend
            'label': 'Mamdani',
            // each array needs to match the length of the breakpoints
            'colors': ['#eff3ff', '#c6dbef', '#9ecae1', '#6baed6', '#3182bd', '#08519c']
        },
        {
            'id': 'Cuomo',
            'label': 'Cuomo',
            'colors': ['#edf8e9', '#c7e9c0', '#a1d99b', '#74c476', '#31a354', '#006d2c']
        },
        {
            'id': 'Silwa',
            'label': 'Silwa',
            'colors': ['#feedde', '#fdd0a2', '#fdae6b', '#fd8d3c', '#e6550d', '#a63603']
        },
        {
            'id': 'Admas',
            'label': 'Admas',
            'colors': ['#f2f0f7', '#dadaeb', '#bcbddc', '#9e9ac8', '#756bb1', '#54278f']
        }
    ],
    'other': {
        'label': 'Other',
        'colors': ['#f7f7f7', '#d9d9d9', '#bdbdbd', '#969696', '#636363', '#252525']
    }
}

const DROP_DOWN_OPTIONS = { // make sure to also update selectedFilters
    'predrace': {
        label: 'Racial Demographics',
        options: [
            { value: 'AsianCombo', label: 'Asian' },
            { value: 'BlackCombo', label: 'Black' },
            { value: 'Hispanic', label: 'Hispanic' },
            { value: 'Tie', label: 'Tie' },
            { value: 'WhiteAlone', label: 'White' }
        ]
    },
    'ownrent': {
        label: 'Housing',
        options: [
            { value: 'OwnMajority', label: 'Owner Majority' },
            { value: 'RentMajority', label: 'Renter Majority' }
        ]
    }
}

const SETTINGS = {
    'getColor': (d: any) => {
        const winnerColumn = 'winner25'
        let value: number | null = null;

        if (d[winnerColumn] === 'Tie') {
            return '#bdbdbd';
        }

        // Find the candidate
        const candidate = COLOR_SCALE.candidates.find(c => c.id === d[winnerColumn]);

        if (!candidate) {
            return null;
        }

        // Get the vote share value
        const valueKey = `${d[winnerColumn].toLowerCase()}25`;
        value = d[valueKey]; // this needs to be a percent instead of a full value

        if (value === null || value === undefined) {
            return null;
        }

        // Use d3 to scale the value to a color based on breakpoints
        const colorScale = d3.scaleThreshold()
            .domain(COLOR_SCALE.breakpoints)
            .range(candidate.colors);

        return colorScale(value);
    },
    'promoteId': 'aded',
}

// generate properties.data
const results = JSON.parse(resultsRaw).features.map((d: any) => {
    d.properties.color = SETTINGS.getColor(d.properties)
    return d
});

// state
const hoveredId = ref<string | number | null>(null)
const clickedId = ref<string | number | null>(null)
const selectedFilters = ref({
    predrace: null as string | null,
    ownrent: null as string | null
})
const activeId = computed(() => clickedId.value ?? hoveredId.value)
const hoveredData = computed(() => {
    if (!activeId.value) return null;

    // Find the feature with the matching ID
    const feature = results.find((f: any) => f.properties[SETTINGS.promoteId] === activeId.value);

    if (!feature) return null;

    return {
        id: activeId.value,
        properties: feature.properties
    };
})

function clearClickedId() {
    clickedId.value = null;
}

function clearFilters() {
    selectedFilters.value = {
        predrace: null,
        ownrent: null
    }
}

// helpers and listeners
function transformRequest(url: string, resourceType: string) {
    if (isMapboxURL(url)) {
        return transformMapboxUrl(url, resourceType ?? 'Unknown', MAPBOX_KEY)
    }
    return { url }
}

function buildFilterExpression(): any[] {
    const conditions = [];

    if (selectedFilters.value.predrace) {
        conditions.push(['==', ['get', 'predrace'], selectedFilters.value.predrace]);
    }

    if (selectedFilters.value.ownrent) {
        conditions.push(['==', ['get', 'ownrent'], selectedFilters.value.ownrent]);
    }

    if (conditions.length === 0) {
        return true;
    }

    if (conditions.length === 1) {
        return conditions[0];
    }

    return ['all', ...conditions];
}

onMounted(() => {
    // Init map adnd add json layer
    const map = new Map({
        container: "map",
        style: MAPBOX_STYLE_URL,
        //@ts-expect-error Type 'maplibregl.ResourceType | undefined' is not assignable to type 'string'
        transformRequest,
        center: [-73.9438, 40.710],
        zoom: 10,
        attributionControl: false
    });

    map.addControl(new AttributionControl({ compact: true }), 'bottom-left');
    map.addControl(new (NavigationControl as any)({ showCompass: false }), 'bottom-left');


    // Wait for maps to load before creating the comparison
    Promise.all([
        new Promise(resolve => map.on('load', resolve))
    ]).then(() => {

        // Add data and styles
        map.addSource('map-source', {
            type: 'geojson',
            data: { "type": "FeatureCollection", features: results },
            promoteId: SETTINGS.promoteId
        })


        map.addLayer({
            'id': 'map-fill',
            'type': 'fill',
            'source': 'map-source',
            'layout': {},
            'paint': {
                'fill-color': ["case", ["has", "color"], ["get", "color"], "#cccccc"],
                'fill-opacity': ['case', ["==", ["get", "color"], null], 0, 0.8],
            }
        }, 'county-outline')

        map.addLayer({
            'id': 'map-line',
            'type': 'line',
            'source': 'map-source',
            'layout': {},
            'paint': {
                'line-color': '#0fff',
                'line-width': 3,
                'line-opacity': [
                    'case',
                    ['boolean', ['feature-state', 'hover'], false],
                    1,
                    ['boolean', ['feature-state', 'clicked'], false],
                    1,
                    0
                ]
            }
        }, 'county-outline')

        // Add mask layer that covers non-matching districts
        map.addLayer({
            'id': 'map-mask',
            'type': 'fill',
            'source': 'map-source',
            'layout': {},
            'paint': {
                'fill-color': '#ffffff',
                'fill-opacity': 0.7,
            },
            'filter': ['!', buildFilterExpression()]
        })

        // Watch for filter changes and update the mask layer
        watch(selectedFilters, (newFilters) => {
            map.setFilter('map-mask', ['!', buildFilterExpression()]);
        }, { deep: true })


        // Add hover interactions
        let hoveredBeforeId: string | number | null = null;
        map.on('mousemove', 'map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                if (hoveredBeforeId !== null) {
                    map.setFeatureState(
                        { source: 'map-source', id: hoveredBeforeId },
                        { hover: false }
                    );
                }
                hoveredBeforeId = e.features[0]?.id ?? null
                if (hoveredBeforeId !== null) {
                    if (!clickedId.value) {
                        hoveredId.value = hoveredBeforeId
                    }
                    map.setFeatureState(
                        { source: 'map-source', id: hoveredBeforeId },
                        { hover: true }
                    );
                }

            }
        });

        map.on('mouseleave', 'map-fill', () => {
            if (hoveredBeforeId !== null) {
                map.setFeatureState(
                    { source: 'map-source', id: hoveredBeforeId },
                    { hover: false }
                );
            }
            hoveredBeforeId = null;
            if (!clickedId.value) {
                hoveredId.value = null;
            }
        });

        map.on('click', 'map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                const featureId = e.features[0]?.id ?? null;
                clickedId.value = featureId;
                console.log(e.features[0])
            }
        });


        // Watch for clickedId changes to update feature state
        let previousClickedId: string | number | null = null;
        watch(clickedId, (newId, oldId) => {
            // Clear previous clicked state
            if (previousClickedId !== null) {
                map.setFeatureState(
                    { source: 'map-source', id: previousClickedId },
                    { clicked: false }
                );
            }
            // Set new clicked state
            if (newId !== null) {
                map.setFeatureState(
                    { source: 'map-source', id: newId },
                    { clicked: true }
                );
            }
            previousClickedId = newId;
        });
    });
})

</script>

<template>
    <div id="comparison-container">
        <div class="cuny-logo-wrapper">
            <img :src="cunygclogo" alt="CUNY Logo" class="cuny-logo">
        </div>

        <div class="top-overlay">
            <h2>NYC 2025 General Election: Mayor</h2>
            <h3>Vote share by election district</h3>
            <Legend :COLOR_SCALE="COLOR_SCALE"></Legend>
            
            <div class="filters-section">
                <div v-for="(filterConfig, filterKey) in DROP_DOWN_OPTIONS" :key="filterKey" class="filter-group">
                    <label :for="`${filterKey}-select`">{{ filterConfig.label }}:</label>
                    <select :id="`${filterKey}-select`" v-model="selectedFilters[filterKey]" class="filter-select">
                        <option :value="null">All</option>
                        <option v-for="option in filterConfig.options" :key="option.value" :value="option.value">
                            {{ option.label }}
                        </option>
                    </select>
                </div>
            </div>

            <div class="button-group">
                <button v-if="selectedFilters.predrace || selectedFilters.ownrent" @click="clearFilters" class="clear-button">
                    Clear Filters
                </button>
                    <button v-if="clickedId" @click="clearClickedId" class="clear-button">
                    Clear Selection
                </button>
            </div>
        </div>
        <InfoBox :hoveredData="hoveredData" :COLOR_SCALE="COLOR_SCALE" :idKey="SETTINGS.promoteId" />
        <div id="map" class="map"></div>
    </div>
</template>

<style scoped>
body {
    margin: 0;
    padding: 0;
}

#comparison-container {
    position: relative;
    width: 100vw;
    height: 100vh;
}

.top-overlay {
    position: absolute;
    z-index: 2;
    background-color: white;
    padding: 0.5rem;
    border-radius: 5px;
    margin: 5px;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    max-width: min(20rem, 50%);
}

.top-overlay h2 {
    margin-top: 0;
    font-size: 1.5rem;
    line-height: 0.9;
}

.top-overlay h3 {
    margin-top: 0;
    font-size: 1.2rem;
    color: #222;
}

.filters-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    padding: 0.5rem 0;
    border-top: 1px solid #ddd;
    border-bottom: 1px solid #ddd;
}

.filter-group {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
}

.filter-group label {
    font-size: 0.875rem;
    font-weight: 600;
    color: #333;
}

.filter-select {
    padding: 0.5rem;
    border: 1px solid #999;
    border-radius: 4px;
    font-size: 0.875rem;
    background-color: white;
    cursor: pointer;
    transition: border-color 0.2s;
}

.filter-select:hover {
    border-color: #666;
}

.filter-select:focus {
    outline: none;
    border-color: #0066cc;
    box-shadow: 0 0 0 2px rgba(0, 102, 204, 0.2);
}

.button-group {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
}

.clear-button {
    background-color: #0066cc;
    color: white;
    border: none;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    cursor: pointer;
    font-weight: 500;
    transition: background-color 0.2s;
    font-size: 0.875rem;
}

.clear-button:hover {
    background-color: #0052a3;
}

.cuny-logo-wrapper {
    position: absolute;
    z-index: 2;
    top: 0;
    right: 0;
    padding: 0.5rem;
    background: linear-gradient(135deg, #ffffff1e 0%, #dfdfdf41 100%);
}

.cuny-logo {
    height: 2.5rem;
    display: block;
}

.map {
    position: absolute;
    top: 0;
    bottom: 0;
    width: 100%;
}

:deep(.year-label) {
    position: absolute;
    top: 50%;
    transform: translateY(1.5rem);
    background: rgba(0, 0, 0, 0.4);
    padding: 0rem 0.5rem;
    border-radius: 4px;
    font-weight: bold;
    font-size: 1.25rem;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
    pointer-events: none;
}

:deep(.year-label-left) {
    left: -4rem;
}

:deep(.year-label-right) {
    right: -4rem;
}
</style>
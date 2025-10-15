<!-- eslint-disable @typescript-eslint/no-explicit-any -->

<script setup lang="ts">
import { Map, NavigationControl } from 'maplibre-gl';
// @ts-expect-error Could not find a declaration file
import Compare from '@maplibre/maplibre-gl-compare'
import 'maplibre-gl/dist/maplibre-gl.css'
import '@maplibre/maplibre-gl-compare/dist/maplibre-gl-compare.css'
import { computed, onMounted, ref, useTemplateRef, watch } from 'vue';
import { isMapboxURL, transformMapboxUrl } from './libs/mapbox-transform';

import cunygclogo from './cunygc_logo.png'

import resultsRaw from './stores/results.geojson?raw';
import InfoBox from './InfoBox.vue'
import Legend from './Legend.vue';

// consts
const MAPBOX_KEY = 'pk.eyJ1IjoiY3VueWN1ciIsImEiOiJfQmNSMF9NIn0.uRgbcFeJbw2xyTUZY8gYeA'
const MAPBOX_STYLE_URL = 'mapbox://styles/cunycur/cm2yzn8mp00ox01pa3oxc4syx';
const COLOR_SCALE = [
    {
        pct: 92,
        color: '#034E7B',
        label: "Dem vote share 92%-100%"
    },
    {
        pct: 85,
        color: '#4393C3',
        label: "Dem vote share 85%-92%"
    },
    {
        pct: 63,
        color: '#9ECAE1',
        label: "Dem vote share 63%-85%"
    },
    {
        pct: 45,
        color: '#EFD7FF',
        label: "GOP-leaning vote share 37%-63%"
    },
    {
        pct: 30,
        color: '#FF7F7F',
        label: "GOP vote share 55%-70%"
    },
    {
        pct: 0,
        color: '#E60000',
        label: "GOP vote share 70%-100%"
    }
]

const SETTINGS = {
    'before': {
        'year': 2020,
        'getData': (d: any) => +d.biden20 / +d.totvote20 * 100,
        'promoteId': 'aded24',
        'candidate': 'Biden',
        'opponent': 'Trump',
        'candidateKey': 'biden20',
        'opponentKey': 'trump20',
        'totalKey': 'totvote20',
        'label': 'Dem',
        'opponentLabel': 'GOP'
    },
    'after': {
        'year': 2024,
        'getData': (d: any) => +d.harris24 / +d.totvote24 * 100,
        'promoteId': 'aded24',
        'candidate': 'Harris',
        'opponent': 'Trump',
        'candidateKey': 'harris24',
        'opponentKey': 'trump24',
        'totalKey': 'totvote24',
        'label': 'Dem',
        'opponentLabel': 'GOP'
    }
}
// generate fill expression
const FILL_EXPRESSION = COLOR_SCALE.reduce((rule, breakpoint) => {
    return [...rule, ['>', ['get', 'data'], breakpoint.pct], breakpoint.color]
}, ['case'] as any[])

// fallback color
FILL_EXPRESSION.push('#00000000')

// generate properties.data
const results = JSON.parse(resultsRaw).features.map((d: any) => {
    d.properties.data = SETTINGS.after.getData(d.properties)
    return d
});

const resultsOld = JSON.parse(resultsRaw).features.map((d: any) => {
    d.properties.data = SETTINGS.before.getData(d.properties)
    return d
});

// state

const comparisonContainer = useTemplateRef('comparison-container')
const hoveredId = ref<string | number | null>(null)
const clickedId = ref<string | number | null>(null)
const activeId = computed(() => clickedId.value ?? hoveredId.value)
const hoveredData = computed(() => {
    if (!activeId.value) return null;

    // Find the feature with the matching ID
    const feature = results.find((f: any) => f.properties[SETTINGS.after.promoteId] === activeId.value);

    if (!feature) return null;

    return {
        id: activeId.value,
        properties: feature.properties
    };
})

function clearClickedId() {
    clickedId.value = null;
}
// helpers and listeners

function transformRequest(url: string, resourceType: string) {
    if (isMapboxURL(url)) {
        return transformMapboxUrl(url, resourceType ?? 'Unknown', MAPBOX_KEY)
    }
    return { url }
}

onMounted(() => {
    // Init map adnd add json layer
    const beforeMap = new Map({
        container: "before",
        style: MAPBOX_STYLE_URL,
        //@ts-expect-error Type 'maplibregl.ResourceType | undefined' is not assignable to type 'string'
        transformRequest,
        center: [-73.9438, 40.710],
        zoom: 10,
    });


    const afterMap = new Map({
        container: "after",
        style: MAPBOX_STYLE_URL,
        //@ts-expect-error Type 'maplibregl.ResourceType | undefined' is not assignable to type 'string'
        transformRequest,
        center: [-73.9438, 40.710],
        zoom: 10,
    });
    afterMap.addControl(new NavigationControl({
        showZoom: true
    }), 'bottom-right');

    // Wait for maps to load before creating the comparison
    Promise.all([
        new Promise(resolve => beforeMap.on('load', resolve)),
        new Promise(resolve => afterMap.on('load', resolve))
    ]).then(() => {
        new Compare(beforeMap, afterMap, comparisonContainer.value, {
            orientation: 'vertical'
        });

        // Add year labels
        const swiper = comparisonContainer.value?.querySelector('.compare-swiper-vertical');
        if (swiper) {
            const leftLabel = document.createElement('div');
            leftLabel.className = 'year-label year-label-left';
            leftLabel.textContent = SETTINGS.before.year.toString();
            swiper.appendChild(leftLabel);

            const rightLabel = document.createElement('div');
            rightLabel.className = 'year-label year-label-right';
            rightLabel.textContent = SETTINGS.after.year.toString();
            swiper.appendChild(rightLabel);
        }

        // Add data and styles

        beforeMap.addSource('before-source', {
            type: 'geojson',
            data: { "type": "FeatureCollection", features: resultsOld },
            promoteId: SETTINGS.before.promoteId
        })

        afterMap.addSource('after-source', {
            type: 'geojson',
            data: { "type": "FeatureCollection", features: results },
            promoteId: SETTINGS.before.promoteId
        })

        beforeMap.addLayer({
            'id': 'before-map-fill',
            'type': 'fill',
            'source': 'before-source',
            'layout': {},
            'paint': {
                //@ts-expect-error typed
                'fill-color': FILL_EXPRESSION,
                'fill-opacity': ['case', ["==", ["get", "data"], null], 0, 0.8],
            }
        }, 'county-outline')

        beforeMap.addLayer({
            'id': 'before-map-line',
            'type': 'line',
            'source': 'before-source',
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

        afterMap.addLayer({
            'id': 'after-map-fill',
            'type': 'fill',
            'source': 'after-source',
            'layout': {},
            'paint': {
                //@ts-expect-error typed
                'fill-color': FILL_EXPRESSION,
                'fill-opacity': ['case', ["==", ["get", "data"], null], 0, 0.8],
            }
        }, 'county-outline')

        afterMap.addLayer({
            'id': 'after-map-line',
            'type': 'line',
            'source': 'after-source',
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

        // Add hover interactions for before map
        let hoveredBeforeId: string | number | null = null;
        beforeMap.on('mousemove', 'before-map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                if (hoveredBeforeId !== null) {
                    beforeMap.setFeatureState(
                        { source: 'before-source', id: hoveredBeforeId },
                        { hover: false }
                    );
                }
                hoveredBeforeId = e.features[0]?.id ?? null
                if (hoveredBeforeId !== null) {
                    if (!clickedId.value) {
                        hoveredId.value = hoveredBeforeId
                    }
                    beforeMap.setFeatureState(
                        { source: 'before-source', id: hoveredBeforeId },
                        { hover: true }
                    );
                }

            }
        });

        beforeMap.on('mouseleave', 'before-map-fill', () => {
            if (hoveredBeforeId !== null) {
                beforeMap.setFeatureState(
                    { source: 'before-source', id: hoveredBeforeId },
                    { hover: false }
                );
            }
            hoveredBeforeId = null;
            if (!clickedId.value) {
                hoveredId.value = null;
            }
        });

        beforeMap.on('click', 'before-map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                const featureId = e.features[0]?.id ?? null;
                clickedId.value = featureId;
            }
        });

        // Add hover interactions for after map
        let hoveredAfterId: string | number | null = null;
        afterMap.on('mousemove', 'after-map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                if (hoveredAfterId !== null) {
                    afterMap.setFeatureState(
                        { source: 'after-source', id: hoveredAfterId },
                        { hover: false }
                    );
                }
                hoveredAfterId = e.features[0]?.id ?? null;
                if (hoveredAfterId !== null) {
                    if (!clickedId.value) {
                        hoveredId.value = hoveredAfterId
                    }
                    afterMap.setFeatureState(
                        { source: 'after-source', id: hoveredAfterId },
                        { hover: true }
                    );
                }

            }
        });

        afterMap.on('mouseleave', 'after-map-fill', () => {
            if (hoveredAfterId !== null) {
                afterMap.setFeatureState(
                    { source: 'after-source', id: hoveredAfterId },
                    { hover: false }
                );
            }
            hoveredAfterId = null;
            if (!clickedId.value) {
                hoveredId.value = null;
            }
        });

        afterMap.on('click', 'after-map-fill', (e) => {
            if (e.features && e.features.length > 0) {
                const featureId = e.features[0]?.id ?? null;
                clickedId.value = featureId;
            }
        });

        // Watch for clickedId changes to update feature state
        let previousClickedId: string | number | null = null;
        watch(clickedId, (newId, oldId) => {
            // Clear previous clicked state
            if (previousClickedId !== null) {
                beforeMap.setFeatureState(
                    { source: 'before-source', id: previousClickedId },
                    { clicked: false }
                );
                afterMap.setFeatureState(
                    { source: 'after-source', id: previousClickedId },
                    { clicked: false }
                );
            }
            // Set new clicked state
            if (newId !== null) {
                beforeMap.setFeatureState(
                    { source: 'before-source', id: newId },
                    { clicked: true }
                );
                afterMap.setFeatureState(
                    { source: 'after-source', id: newId },
                    { clicked: true }
                );
            }
            previousClickedId = newId;
        });
    });
})

</script>

<template>
    <div id="comparison-container" ref="comparison-container">
        <div class="cuny-logo-wrapper">
            <img :src="cunygclogo" alt="CUNY Logo" class="cuny-logo">
        </div>

        <div class="top-overlay">
            <h2>Vote share by election district</h2>
            <Legend :COLOR_SCALE="COLOR_SCALE"></Legend>
            <button v-if="clickedId" @click="clearClickedId" class="clear-button">
                Clear Selection
            </button>
        </div>
        <InfoBox :hovered-data="hoveredData" :before-settings="SETTINGS.before" :after-settings="SETTINGS.after"
            :id-key="SETTINGS.after.promoteId" />
        <div id="before" class="map"></div>
        <div id="after" class="map"></div>
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
    box-shadow: 2px 2px 8px darkgray;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    max-width: 15rem;
}

.top-overlay h2{
    margin-top: 0;
    font-size: 1.5rem;
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
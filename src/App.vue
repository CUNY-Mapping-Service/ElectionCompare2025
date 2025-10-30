<!-- eslint-disable @typescript-eslint/no-explicit-any -->

<script setup lang="ts">
import { AttributionControl, Map as MaplibreMap, NavigationControl } from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css'
import '@maplibre/maplibre-gl-compare/dist/maplibre-gl-compare.css'
import { computed, onMounted, ref, watch } from 'vue';
import { isMapboxURL, transformMapboxUrl } from './libs/mapbox-transform';
import * as d3 from "d3";
import { Index } from 'flexsearch';

import cunygclogo from './cunygc_logo.png'

import NYED_GEOM from './stores/nyed25c.json';
import FILTER_DATA_RAW from './stores/filterdata.csv?raw'
import METADATA from './stores/metadata.json'
import RESULTS_DATA_RAW from './stores/test25results.csv?raw'

import InfoBox from './InfoBox.vue'
import Legend from './Legend.vue';

// consts
const MAPBOX_KEY = 'pk.eyJ1IjoiY3VueWN1ciIsImEiOiJfQmNSMF9NIn0.uRgbcFeJbw2xyTUZY8gYeA'
const MAPBOX_STYLE_URL = 'mapbox://styles/cunycur/cm2yzn8mp00ox01pa3oxc4syx';
const COLOR_SCALE = {
    'breakpoints': [
        5, 10, 25, 50, 75, 100
    ],
    'total_id': 'gen25tot',
    'candidates': [
        {
            'id': 'gen25zm',
            // label for legend, needs to match win25
            'label': 'Zohran Mamdani',
            // each array needs to match the length of the breakpoints
            'colors': ['#eff3ff', '#c6dbef', '#9ecae1', '#6baed6', '#3182bd', '#08519c']
        },
        {
            'id': 'gen25ac',
            'label': 'Andrew Cuomo',
            'colors': ['#edf8e9', '#c7e9c0', '#a1d99b', '#74c476', '#31a354', '#006d2c']
        },
        {
            'id': 'gen25cs',
            'label': 'Curtis Sliwa',
            'colors': ['#feedde', '#fdd0a2', '#fdae6b', '#fd8d3c', '#e6550d', '#a63603']
        },
        {
            'id': 'gen25ea',
            'label': 'Eric Adams',
            'colors': ['#f2f0f7', '#dadaeb', '#bcbddc', '#9e9ac8', '#756bb1', '#54278f']
        }
    ],
    'other': {
        'label': 'Other',
        'colors': ['#f7f7f7', '#d9d9d9', '#bdbdbd', '#969696', '#636363', '#252525']
    }
}

// Build filter options from METADATA
const FILTER_OPTIONS = METADATA
    .filter((item: any) => item.isFilter)
    .map((item: any) => ({
        column: item.column,
        label: item.label,
        short_label: item.short_label || item.label
    }));

const SETTINGS = {
    'getColor': (d: any) => {
        const winnerColumn = 'win25'
        let value: number | null = null;

        if (d[winnerColumn] === 'Tie') {
            return '#bdbdbd';
        }

        // Find the candidate column using the label
        // win25 is "Curtis Sliwa" get gen25cs
        const candidate = COLOR_SCALE.candidates.find(c => c.label === d[winnerColumn]);
        if (!candidate) {
            return null;
        }

        // Get the vote share value (use percentage)
        const percentKey = `${candidate.id}_pct`
        value = d[percentKey];

        if (value === null || value === undefined) {
            return null;
        }

        // Use d3 to scale the value to a color based on breakpoints
        const colorScale = d3.scaleThreshold()
            .domain(COLOR_SCALE.breakpoints)
            // @ts-ignore  Type 'string' is not assignable to type 'number'
            .range(candidate.colors);

        return colorScale(value);
    },
    'promoteId': 'ElectDist',
}

// generate properties.data
const parsedFilterData = d3.csvParse(FILTER_DATA_RAW, d3.autoType) as any[];
const FILTER_PROPERTIES = new Map(
    parsedFilterData.map(({ ElectDist, ...rest }) => [String(ElectDist), rest])
);

const parsedResultsData = d3.csvParse(RESULTS_DATA_RAW, d3.autoType) as any[];
const RESULTS_PROPERTIES = new Map(
    parsedResultsData.map(({ ElectDist, ...rest }) => [String(ElectDist), rest])
);

const features = NYED_GEOM.features.map((d: any) => {
    const electDist = String(d.properties.ElectDist);
    const filterProps = FILTER_PROPERTIES.get(electDist) || {};
    const resultsProps = RESULTS_PROPERTIES.get(electDist) || {};

    d.properties = {
        ...d.properties,
        ...filterProps,
        ...resultsProps
    };

    // Calculate percentages for 2025
    const gen25total = d.properties.gen25tot || 0;
    if (gen25total > 0) {
        d.properties.gen25zm_pct = ((d.properties.gen25zm || 0) / gen25total) * 100;
        d.properties.gen25ac_pct = ((d.properties.gen25ac || 0) / gen25total) * 100;
        d.properties.gen25cs_pct = ((d.properties.gen25cs || 0) / gen25total) * 100;
        d.properties.gen25ea_pct = ((d.properties.gen25ea || 0) / gen25total) * 100;
    } else {
        d.properties.gen25zm_pct = 0;
        d.properties.gen25ac_pct = 0;
        d.properties.gen25cs_pct = 0;
        d.properties.gen25ea_pct = 0;
    }

    // Calculate percentages for 2021
    const gen21total = d.properties.gen21tot || 0;
    if (gen21total > 0) {
        d.properties.gen21ea_pct = ((d.properties.gen21ea || 0) / gen21total) * 100;
        d.properties.gen21cs_pct = ((d.properties.gen21cs || 0) / gen21total) * 100;
        d.properties.gen21othr_pct = ((d.properties.gen21othr || 0) / gen21total) * 100;
    } else {
        d.properties.gen21ea_pct = 0;
        d.properties.gen21cs_pct = 0;
        d.properties.gen21othr_pct = 0;
    }

    d.properties.color = SETTINGS.getColor(d.properties);
    return d;
});

// state
const hoveredId = ref<string | number | null>(null)
const clickedId = ref<string | number | null>(null)
const selectedFilters = ref<Array<{ column: string; label: string; short_label: string }>>([])
const searchQuery = ref('')
const searchResults = ref<Array<{ column: string; label: string; short_label: string }>>([])
const showSearchResults = ref(false)
const showCityCouncil = ref(false)
const showNYCHA = ref(false)

// Initialize FlexSearch index
const searchIndex = new Index({
    tokenize: 'forward',
    cache: true
})

// Index all filter options with their labels and keyterms
FILTER_OPTIONS.forEach((option: any, index: number) => {
    const metadata = METADATA.find((m: any) => m.column === option.column)
    const keyterms = metadata?.keyterms || []

    // Combine label and keyterms for comprehensive search
    const searchText = [option.label, ...keyterms].join(' ')
    searchIndex.add(index, searchText)
})

const activeId = computed(() => clickedId.value ?? hoveredId.value)
const hoveredData = computed(() => {
    if (!activeId.value) return null;

    // Find the feature with the matching ID
    const feature = features.find((f: any) => f.properties[SETTINGS.promoteId] === activeId.value);

    if (!feature) return null;

    return {
        id: activeId.value,
        properties: feature.properties
    };
})

// Compute filtered districts based on selected filters
const filteredDistricts = computed(() => {
    if (selectedFilters.value.length === 0) {
        return [];
    }

    return features.filter((feature: any) => {
        // Check if feature matches all selected filters
        return selectedFilters.value.every(filter => {
            return feature.properties[filter.column] === 1;
        });
    });
})

function clearClickedId() {
    clickedId.value = null;
    hoveredId.value = null;
}

function clearFilters() {
    selectedFilters.value = []
}

function addFilter(filter: { column: string; label: string; short_label: string }) {
    // Check if filter already exists
    const exists = selectedFilters.value.some(f => f.column === filter.column)
    if (!exists) {
        selectedFilters.value.push(filter)
    }
    // Clear search
    searchQuery.value = ''
    searchResults.value = []
    showSearchResults.value = false
}

function removeFilter(column: string) {
    selectedFilters.value = selectedFilters.value.filter(f => f.column !== column)
}

function performSearch() {
    if (!searchQuery.value.trim()) {
        // Show all results when query is empty
        searchResults.value = FILTER_OPTIONS
        showSearchResults.value = true
        return
    }
    const results = searchIndex.search(searchQuery.value)
    searchResults.value = results.map((index: any) => FILTER_OPTIONS[index]).filter(Boolean) as Array<{ column: string; label: string; short_label: string }>
    showSearchResults.value = true
}

function handleBlur() {
    window.setTimeout(() => {
        showSearchResults.value = false
    }, 200)
}

// Watch search query
watch(searchQuery, () => {
    performSearch()
})

function transformRequest(url: string, resourceType: string) {
    if (isMapboxURL(url)) {
        return transformMapboxUrl(url, resourceType ?? 'Unknown', MAPBOX_KEY)
    }
    return { url }
}

function buildFilterExpression(): any[] {
    if (selectedFilters.value.length === 0) {
        return ['all']
    }

    const conditions = selectedFilters.value.map(filter => {
        // All filter columns in METADATA are boolean (0 or 1)
        return ['==', ['get', filter.column], 1]
    })

    return ['all', ...conditions];
}

onMounted(() => {
    // Init map adnd add json layer
    const map = new MaplibreMap({
        container: "map",
        style: MAPBOX_STYLE_URL,
        //@ts-expect-error Type 'maplibregl.ResourceType | undefined' is not assignable to type 'string'
        transformRequest,
        center: [-73.9438, 40.710],
        zoom: 10,
        attributionControl: false
    });

    map.addControl(new AttributionControl({ compact: true, customAttribution: '' }), 'bottom-left');
    map.addControl(new (NavigationControl as any)({ showCompass: false }), 'bottom-left');


    // Wait for maps to load before creating the comparison
    Promise.all([
        new Promise(resolve => map.on('load', resolve))
    ]).then(() => {

        // Add overlay sources
        map.addSource('citycouncil-source', {
            type: 'vector',
            tiles: ['https://www.urbanresearchmaps.org/tiles/common.wrm_citycouncil.geom/{z}/{x}/{y}'],
            minzoom: 0,
            maxzoom: 14
        });

        map.addSource('nycha-source', {
            type: 'vector',
            tiles: ['https://www.urbanresearchmaps.org/tiles/common.wrm_nycha.geom/{z}/{x}/{y}'],
            minzoom: 0,
            maxzoom: 14
        });

        // Add data and styles
        map.addSource('map-source', {
            type: 'geojson',
            data: { "type": "FeatureCollection", features: features },
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
        // @ts-ignore
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

        // Add City Council layers
        map.addLayer({
            'id': 'citycouncil-fill',
            'type': 'fill',
            'source': 'citycouncil-source',
            'source-layer': 'common.wrm_citycouncil.geom',
            'layout': {
                'visibility': 'none'
            },
            'paint': {
                'fill-color': 'transparent',
                'fill-outline-color': '#000000'
            }
        });

        map.addLayer({
            'id': 'citycouncil-line',
            'type': 'line',
            'source': 'citycouncil-source',
            'source-layer': 'common.wrm_citycouncil.geom',
            'layout': {
                'visibility': 'none'
            },
            'paint': {
                'line-color': '#000000',
                'line-width': 2
            }
        });

        map.addLayer({
            'id': 'citycouncil-label',
            'type': 'symbol',
            'source': 'citycouncil-source',
            'source-layer': 'common.wrm_citycouncil.geom',
            'layout': {
                'visibility': 'none',
                'text-field': ['get', 'districtid'],
                'text-font': ['Open Sans Bold', 'Arial Unicode MS Bold'],
                'text-size': 14
            },
            'paint': {
                'text-color': '#000000',
                'text-halo-color': '#ffffff',
                'text-halo-width': 2
            }
        });

        // Add NYCHA layers
        map.addLayer({
            'id': 'nycha-fill',
            'type': 'fill',
            'source': 'nycha-source',
            'source-layer': 'common.wrm_nycha.geom',
            'layout': {
                'visibility': 'none'
            },
            'paint': {
                'fill-color': '#ff6b6b',
                'fill-opacity': 0.3
            }
        });

        map.addLayer({
            'id': 'nycha-line',
            'type': 'line',
            'source': 'nycha-source',
            'source-layer': 'common.wrm_nycha.geom',
            'layout': {
                'visibility': 'none'
            },
            'paint': {
                'line-color': '#c92a2a',
                'line-width': 1.5
            }
        });

        // Watch for filter changes and update the mask layer
        watch(selectedFilters, (newFilters) => {
            // @ts-ignore
            map.setFilter('map-mask', ['!', buildFilterExpression()]);
        }, { deep: true })

        // Watch for City Council layer toggle
        watch(showCityCouncil, (newValue) => {
            const visibility = newValue ? 'visible' : 'none';
            map.setLayoutProperty('citycouncil-fill', 'visibility', visibility);
            map.setLayoutProperty('citycouncil-line', 'visibility', visibility);
            map.setLayoutProperty('citycouncil-label', 'visibility', visibility);
        });

        // Watch for NYCHA layer toggle
        watch(showNYCHA, (newValue) => {
            const visibility = newValue ? 'visible' : 'none';
            map.setLayoutProperty('nycha-fill', 'visibility', visibility);
            map.setLayoutProperty('nycha-line', 'visibility', visibility);
        });


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
    <div id="main">
        <div class="comparison-container">
            <div class="details">
                <h2>NYC 2025 General Election: Mayor</h2>
                <h3>Vote share by election district</h3>

                <div class="filters-section">
                    <label for="filter-search">Show Election Districts based on:</label>
                    <div class="search-container">
                        <input type="text" v-model="searchQuery"
                            placeholder="Search filters (e.g., 'renters', 'income')..." class="filter-search"
                            @focus="performSearch()" @blur="handleBlur" autocomplete="off" />

                        <div v-if="showSearchResults && searchResults.length > 0" class="search-results">
                            <div v-for="result in searchResults" :key="result.column" class="search-result-item"
                                @click="addFilter(result)" :title="result.label">
                                {{ result.short_label }}
                            </div>
                        </div>

                        <div v-if="showSearchResults && searchQuery && searchResults.length === 0"
                            class="search-results">
                            <div class="search-result-item no-results">
                                No filters found
                            </div>
                        </div>
                    </div>

                    <div v-if="selectedFilters.length > 0" class="selected-filters">
                        <div v-for="filter in selectedFilters" :key="filter.column" class="filter-pill"
                            :title="filter.label">
                            <span class="filter-pill-text">{{ filter.short_label }}</span>
                            <button @click="removeFilter(filter.column)" class="filter-pill-close"
                                aria-label="Remove filter">
                                ×
                            </button>
                        </div>
                    </div>
                </div>

                <!-- <div class="button-group">
                    <button v-if="selectedFilters.length > 0" @click="clearFilters" class="clear-button">
                        Clear All Filters
                    </button>
                </div> -->
                <InfoBox :hoveredData="hoveredData" :COLOR_SCALE="COLOR_SCALE" :idKey="SETTINGS.promoteId"
                    :filteredFeatures="filteredDistricts" :selectedFilters="selectedFilters" :allFeatures="features"
                    :metadata="METADATA" @close="clearClickedId" />
            </div>
            <div class="map-container">
                <div id="map" class="map"></div>
                <div class="cuny-logo-wrapper">
                    <img :src="cunygclogo" alt="CUNY Logo" class="cuny-logo">
                </div>
                <Legend :COLOR_SCALE="COLOR_SCALE" :showCityCouncil="showCityCouncil" :showNYCHA="showNYCHA"
                    @update:showCityCouncil="showCityCouncil = $event" @update:showNYCHA="showNYCHA = $event" />
            </div>
        </div>

    </div>
</template>

<style scoped>
body {
    margin: 0;
    padding: 0;
}

#main {
    position: relative;
    width: 100vw;
    height: 100vh;
}

.comparison-container {
    display: flex;
    flex-direction: column;
    height: 100%;
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    font-family: Roboto, Helvetica, Arial, sans-serif;
}

@media (min-width: 768px) {
    .comparison-container {
        flex-direction: row;
    }
}

.details {
    width: 100%;
    height: 60%;
    overflow: auto;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
    color: #1f2937;
    display: flex;
    flex-direction: column;
    padding: 1rem;
    background-color: white;
}

@media (min-width: 768px) {
    .details {
        height: 100%;
        width: 25rem;
    }
}

.map-container {
    position: relative;
    flex: 1;
    order: -1;
}

@media (min-width: 768px) {
    .map-container {
        order: 0;
    }
}

.top-overlay {
    margin: 5px;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    max-width: min(25rem, 50%);
}

.map {
    width: 100%;
    height: 100%;
}

.details h2 {
    margin: 0;
    font-size: 1.5rem;
}

.details h3 {
    margin-top: 0;
    font-size: 0.8rem;
    color: #222;
}

.filters-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    padding: 1rem 0;
}

.filters-section label {
    font-size: 0.875rem;
    font-weight: 600;
    color: #333;
}

.search-container {
    position: relative;
}

.filter-search {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #999;
    border-radius: 4px;
    font-size: 0.875rem;
    background-color: white;
    transition: border-color 0.2s;
    box-sizing: border-box;
}

.filter-search:hover {
    border-color: #666;
}

.filter-search:focus {
    outline: none;
    border-color: #0066cc;
    box-shadow: 0 0 0 2px rgba(0, 102, 204, 0.2);
}

.search-results {
    position: absolute;
    top: 100%;
    left: 0;
    right: 0;
    background: white;
    border: 1px solid #ddd;
    border-radius: 4px;
    margin-top: 4px;
    max-height: 200px;
    overflow-y: auto;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    z-index: 10;
}

.search-result-item {
    padding: 0.5rem;
    cursor: pointer;
    font-size: 0.875rem;
    transition: background-color 0.15s;
}

.search-result-item:hover {
    background-color: #f0f0f0;
}

.search-result-item.no-results {
    color: #666;
    cursor: default;
    font-style: italic;
}

.search-result-item.no-results:hover {
    background-color: white;
}

.selected-filters {
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
    margin-top: 0.5rem;
}

.filter-pill {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    background-color: gray;
    color: white;
    padding: 0.1rem 0.75rem;
    border-radius: 10px;
    font-size: 0.875rem;
    font-weight: 500;
}

.filter-pill-text {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: wrap;
}

.filter-pill-close {
    background: none;
    border: none;
    color: white;
    font-size: 1.25rem;
    line-height: 1;
    cursor: pointer;
    padding: 0;
    margin: 0;
    width: 20px;
    height: 20px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: background-color 0.15s;
}

.filter-pill-close:hover {
    background-color: rgba(255, 255, 255, 0.2);
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
    padding: 0.2rem;
    pointer-events: none;
}

.cuny-logo {
    height: 2.5rem;
    display: block;
    border-radius: 6px;
    background: linear-gradient(135deg, #ffffff1e 0%, #dfdfdf41 100%);
    padding: 0.1rem;
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
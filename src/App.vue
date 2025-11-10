<!-- eslint-disable @typescript-eslint/no-explicit-any -->

<script setup lang="ts">
import { AttributionControl, Map as MaplibreMap, NavigationControl } from 'maplibre-gl';
import 'maplibre-gl/dist/maplibre-gl.css'
// import '@maplibre/maplibre-gl-compare/dist/maplibre-gl-compare.css'
import { computed, onMounted, ref, watch } from 'vue';
import { isMapboxURL, transformMapboxUrl } from './libs/mapbox-transform';
import * as d3 from "d3";
import { Index } from 'flexsearch';

import cunygclogo from './cunygc_logo_cur.png'


import NYED_GEOM from './stores/nyed25.json';
import FILTER_DATA_RAW from './stores/filterdata.csv?raw'
import METADATA from './stores/metadata.json'
const RESULTS_CSV_FILE = 'unofficial25results.csv'

import AddressSearch from './AddressSearch.vue';
import InfoBox from './InfoBox.vue'
import Legend from './Legend.vue';

// consts
const HIDE_ED_VOTE_COUNT_THRESHOLD = 4 // i.e., less than or equal to 4 or less than 5
const MAPBOX_KEY = 'pk.eyJ1IjoiY3VueWN1ciIsImEiOiJfQmNSMF9NIn0.uRgbcFeJbw2xyTUZY8gYeA'
const MAPBOX_STYLE_URL = 'mapbox://styles/cunycur/cm2yzn8mp00ox01pa3oxc4syx';
const COLOR_SCALE = {
    'breakpoints': [
        1, /* 5, 10, */ 25, 50, 75/* , 100 */
    ],
    'total_id': 'gen25tot',
    'candidates': [
        {
            'id': 'gen25zm',
            // label for legend, needs to match win25
            'label': 'Zohran Mamdani',
            // each array needs to match the length of the breakpoints

            //Blue palette via ColorBrewer
            'colors': ['#fff', '#bdd7e7', '#6baed6', '#3182bd', '#08519c']
        },
        {
            'id': 'gen25ac',
            'label': 'Andrew Cuomo',
            //Green palette via ColorBrewer
            // 'colors': ['#edf8e9', '#c7e9c0', '#a1d99b', '#74c476', '#31a354', '#006d2c']
            //YellowGreen palette via ColorBrewer
            'colors': ['#fff', '#ffffcc', '#addd8e', '#31a354', '#006837']
        },
        {
            'id': 'gen25cs',
            'label': 'Curtis Sliwa',
            //Red palette via ColorBrewer
            'colors': ['#fff', '#fcae91', '#fb6a4a', '#de2d26', '#a50f15']
        },
        {
            'id': 'gen25ea',
            'label': 'Eric Adams',
            //Purple palette via ColorBrewer
            // 'colors': ['#fff','#cbc9e2','#9e9ac8','#756bb1','#54278f']
            //Purple palette via Material Design https://m2.material.io/design/color/the-color-system.html#tools-for-picking-colors
            // 'colors': ['#F3E5F5', '#E1BEE7', '#CE93D8', '#AB47BC', '#8E24AA', '#6A1B9A']
            'colors': ['#fff', '#F3E5F5', '#CE93D8', '#9C27B0', /* '#8E24AA', */ '#6A1B9A']
        }
    ],
    'other': {
        'id': 'gen25othr',
        'label': 'Other',
        //Gray palette via ColorBrewer
        'colors': ['#fff', '#d9d9d9', '#bdbdbd', /* '#969696', */ '#636363', '#252525']
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
        const candidate = COLOR_SCALE.candidates.find(c => c.id === d[winnerColumn]);
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
    'promoteId': 'aded25',
}

// generate properties.data
const parsedFilterData = d3.csvParse(FILTER_DATA_RAW, d3.autoType) as any[];
const FILTER_PROPERTIES = new Map(
    parsedFilterData.map(({ aded25, ...rest }) => [String(aded25), rest])
);

// state
const hoveredId = ref<string | number | null>(null)
const clickedId = ref<string | number | null>(null)
const selectedFilters = ref<Array<{ column: string; label: string; short_label: string }>>([])
const searchQuery = ref('')
const searchResults = ref<Array<{ column: string; label: string; short_label: string }>>([])
const showSearchResults = ref(false)
const showCityCouncil = ref(false)
const showNYCHA = ref(false)
const showSubway = ref(false)
const features = ref<any[]>([])


// Map state for URL sync
const mapCenter = ref<[number, number]>([-73.9769, 40.72103])
const mapZoom = ref<number>(10.5)
/* const mapCenter = ref<[number, number]>([-73.9438, 40.710])
const mapZoom = ref<number>(10) */
let mapInstance: MaplibreMap | null = null

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
    const feature = features.value.find((f: any) => f.properties[SETTINGS.promoteId] === activeId.value);

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

    return features.value.filter((feature: any) => {
        // Check if feature matches all selected filters
        return selectedFilters.value.every(filter => {
            return feature.properties[filter.column] === 1;
        });
    });
})

// URL State Management
function updateURL() {
    const params = new URLSearchParams()

    // Add filters
    if (selectedFilters.value.length > 0) {
        params.set('filters', selectedFilters.value.map(f => f.column).join(','))
    }

    // Add map position
    if (mapInstance) {
        const center = mapInstance.getCenter()
        const zoom = mapInstance.getZoom()
        params.set('lat', center.lat.toFixed(5))
        params.set('lng', center.lng.toFixed(5))
        params.set('zoom', zoom.toFixed(2))
    }

    // Add selected ED
    if (clickedId.value) {
        params.set('ed', String(clickedId.value))
    }

    const newURL = params.toString() ? `${window.location.pathname}?${params.toString()}` : window.location.pathname
    window.history.replaceState({}, '', newURL)
}

function loadFromURL() {
    const params = new URLSearchParams(window.location.search)
    // Load filters
    const filtersParam = params.get('filters')
    if (filtersParam) {
        const filterColumns = filtersParam.split(',')
        selectedFilters.value = FILTER_OPTIONS.filter(opt => filterColumns.includes(opt.column))
    }

    // Load map position
    const lat = params.get('lat')
    const lng = params.get('lng')
    const zoom = params.get('zoom')

    if (lat && lng && zoom) {
        mapCenter.value = [parseFloat(lng), parseFloat(lat)]
        mapZoom.value = parseFloat(zoom)
    }

    const ed = params.get('ed')
    if (ed) {
        const feature = features.value.find((f: any) => String(f.properties[SETTINGS.promoteId]) === ed)
        if (feature) {
            clickedId.value = feature.properties[SETTINGS.promoteId]
        }
    }
}

function clearClickedId() {
    clickedId.value = null;
    hoveredId.value = null;
    updateURL()
}

function clearFilters() {
    selectedFilters.value = []
    updateURL()
}

function addFilter(filter: { column: string; label: string; short_label: string }) {
    // Check if filter already exists
    const exists = selectedFilters.value.some(f => f.column === filter.column)
    if (!exists) {
        selectedFilters.value.push(filter)
        updateURL()
    }
    // Clear search
    searchQuery.value = ''
    searchResults.value = []
    showSearchResults.value = false
}

function removeFilter(column: string) {
    selectedFilters.value = selectedFilters.value.filter(f => f.column !== column)
    updateURL()
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

// Watch for clickedId changes to update URL
watch(clickedId, () => {
    updateURL()
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

onMounted(async () => {
    // Fetch the results data
    const parsedResultsData = await d3.csv(`${import.meta.env.BASE_URL}${RESULTS_CSV_FILE}`, d3.autoType) as any[];
    const RESULTS_PROPERTIES = new Map(
        parsedResultsData.map(({ aded25, ...rest }) => [String(aded25), rest])
    );

    // Process features with the fetched data
    const processed_features = NYED_GEOM.features.map((d: any) => {
        const aded25 = String(d.properties.aded25);
        const filterProps = FILTER_PROPERTIES.get(aded25) || {};
        const resultsProps = RESULTS_PROPERTIES.get(aded25) || {};

        d.properties = {
            ...d.properties,
            ...filterProps,
            ...resultsProps
        };

        // Calculate percentages for 2025, round first
        const gen25total = Math.round(d.properties.gen25tot || 0);
        d.properties.gen25zm = Math.round(d.properties.gen25zm || 0);
        d.properties.gen25ac = Math.round(d.properties.gen25ac || 0);
        d.properties.gen25cs = Math.round(d.properties.gen25cs || 0);
        d.properties.gen25ea = Math.round(d.properties.gen25ea || 0);
        d.properties.gen25othr = Math.round(d.properties.gen25othr || 0);

        if (gen25total > 0) {
            d.properties.gen25zm_pct = (d.properties.gen25zm / gen25total) * 100;
            d.properties.gen25ac_pct = (d.properties.gen25ac / gen25total) * 100;
            d.properties.gen25cs_pct = (d.properties.gen25cs / gen25total) * 100;
            d.properties.gen25ea_pct = (d.properties.gen25ea / gen25total) * 100;
            d.properties.gen25othr_pct = (d.properties.gen25othr / gen25total) * 100;
        }

        // Calculate percentages for 2021
        const gen21total = Math.round(d.properties.gen21tot || 0);
        d.properties.gen21ea = Math.round(d.properties.gen21ea || 0);
        d.properties.gen21cs = Math.round(d.properties.gen21cs || 0);
        d.properties.gen21othr = Math.round(d.properties.gen21othr || 0);

        if (gen21total > 0) {
            d.properties.gen21ea_pct = (d.properties.gen21ea / gen21total) * 100;
            d.properties.gen21cs_pct = (d.properties.gen21cs / gen21total) * 100;
            d.properties.gen21othr_pct = (d.properties.gen21othr / gen21total) * 100;
        }

        d.properties.color = SETTINGS.getColor(d.properties);
        return d;
    });

    // modify geometry those on and below the threshold so they are "hidden"
    features.value = processed_features.map(featureItem => {
        if (featureItem.properties.gen21total <= HIDE_ED_VOTE_COUNT_THRESHOLD || featureItem.properties.gen25tot <= HIDE_ED_VOTE_COUNT_THRESHOLD) {
            featureItem.geometry.coordinates = [[[0, 0], [0, 0], [0, 0], [0, 0], [0, 0]]]
        }
        return featureItem
    })


    // Load state from URL after data is loaded
    loadFromURL()


    // Init map and add json layer
    const map = new MaplibreMap({
        container: "map",
        style: MAPBOX_STYLE_URL,
        //@ts-expect-error Type 'maplibregl.ResourceType | undefined' is not assignable to type 'string'
        transformRequest,
        center: mapCenter.value,
        zoom: mapZoom.value,
        attributionControl: false
    });

    mapInstance = map

    map.addControl(new AttributionControl({ compact: true, customAttribution: '' }), 'bottom-left');
    map.addControl(new (NavigationControl as any)({ showCompass: false }), 'bottom-left');

    // Update URL when map moves
    map.on('moveend', () => {
        updateURL()
    })

    // Wait for maps to load before creating the comparison
    Promise.all([
        new Promise(resolve => map.on('load', resolve))
    ]).then(async () => {
        // Load sprites
        const image = await map.loadImage(`${import.meta.env.BASE_URL}sprites/circle.png`);
        if (!map.hasImage('sub-circle')) map.addImage('sub-circle', image.data, { "sdf": true });

        // Add overlay sources
        map.addSource('subway-source', {
            type: 'vector',
            tiles: ['https://www.urbanresearchmaps.org/tiles/common.imagenyc_subwayroutes.geom/{z}/{x}/{y}'],
            minzoom: 0,
            maxzoom: 14
        });

        map.addSource('citycouncil-source', {
            type: 'vector',
            tiles: ['https://www.urbanresearchmaps.org/tiles/common.wrm_citycouncil.geom,common.wrm_citycouncil.geom_pt/{z}/{x}/{y}'],
            minzoom: 0,
            maxzoom: 14
        });

        map.addSource('nycha-source', {
            type: 'vector',
            tiles: ['https://www.urbanresearchmaps.org/tiles/common.nycha.geom,common.nycha.geom_pt/{z}/{x}/{y}'],
            minzoom: 0,
            maxzoom: 14
        });

        // Add data and styles
        map.addSource('map-source', {
            type: 'geojson',
            data: { "type": "FeatureCollection", features: features.value },
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
        }, 'county-outline')

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
                'line-color': '#4d4d4d',
                'line-width': 2
            }
        });

        map.addLayer({
            'id': 'citycouncil-label',
            'type': 'symbol',
            'source': 'citycouncil-source',
            'source-layer': 'common.wrm_citycouncil.geom_pt',
            'layout': {
                'visibility': 'none',
                'text-field': ['get', 'districtid'],
                'text-font': ['Open Sans Bold', 'Arial Unicode MS Bold'],
                'text-size': 12
            },
            'paint': {
                'text-color': '#4d4d4d',
                'text-halo-color': '#ffffff',
                'text-halo-width': 1
            }
        });


        // Add NYC Subway layers
        map.addLayer({
            'id': 'subway-line',
            'type': 'line',
            'source': 'subway-source',
            'source-layer': 'common.imagenyc_subwayroutes.geom',
            'layout': {
                'visibility': 'none'
            },
            "paint": {
                "line-color": [
                    "get",
                    "linecolor"
                ],
                "line-opacity": 1,
                "line-width": 2,
                "line-gap-width": 0
            }
        }, 'boro-outline');

        map.addLayer({
            'id': 'subway-label',
            'type': 'symbol',
            'source': 'subway-source',
            'source-layer': 'common.imagenyc_subwayroutes.geom',
            "layout": {
                "text-field": "{lineid}",
                "symbol-placement": "line",
                "text-size": 12,
                "icon-allow-overlap": true,
                "icon-ignore-placement": false,
                "icon-optional": true,
                "text-optional": false,
                "visibility": "none",
                "icon-image": "sub-circle",
                "text-keep-upright": false,
                "icon-rotation-alignment": "viewport",
                "symbol-avoid-edges": false,
                "symbol-spacing": 250,
                "icon-text-fit": "none",
                "icon-padding": 5,
                "text-rotation-alignment": "viewport",
                "text-pitch-alignment": "viewport",
                "text-allow-overlap": false,
                "text-ignore-placement": false,
                "text-max-width": 12,
                "text-variable-anchor": [
                    "center"
                ],
                "symbol-z-order": "auto",
                "text-anchor": "center",
                "icon-text-fit-padding": [
                    0,
                    0,
                    0,
                    0
                ],
                "text-padding": 0,
                "text-offset": [
                    0,
                    0
                ],
                "text-letter-spacing": 0,
                "text-font": [
                    "literal",
                    [
                        "Arial Unicode MS Bold",
                        "Open Sans Bold"
                    ]
                ],
                "icon-size": 0.25
            },
            "paint": {
                "text-color": [
                    "match",
                    [
                        "get",
                        "lineid"
                    ],
                    "N",
                    "#000",
                    "Q",
                    "#000",
                    "R",
                    "#000",
                    "W",
                    "#000",
                    "Air",
                    "#000",
                    "#fff"
                ],
                "text-halo-color": [
                    "get",
                    "linecolor"
                ],
                "icon-halo-width": 0,
                "icon-halo-blur": 0,
                "text-halo-width": 0,
                "text-halo-blur": 4,
                "icon-translate-anchor": "map",
                "icon-color": [
                    "get",
                    "linecolor"
                ],
                "icon-translate": [
                    0,
                    0
                ],
                "icon-opacity": 1
            }
        }, 'boro-outline');


        // Add NYCHA layers
        map.addLayer({
            'id': 'nycha-fill',
            'type': 'fill',
            'source': 'nycha-source',
            'source-layer': 'common.nycha.geom',
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
            'source-layer': 'common.nycha.geom',
            'layout': {
                'visibility': 'none'
            },
            'paint': {
                'line-color': '#c92a2a',
                'line-width': 1.5
            }
        });

        map.addLayer({
            'id': 'nycha-label',
            'type': 'symbol',
            'source': 'nycha-source',
            'source-layer': 'common.nycha.geom_pt',
            'layout': {
                'visibility': 'none',
                'text-field': ['get', 'name'],
                'text-font': ['Open Sans Regular', 'Arial Unicode MS Regular'],
                "text-size": [
                    "interpolate",
                    [
                        "linear"
                    ],
                    [
                        "zoom"
                    ],
                    0,
                    0,
                    10,
                    0,
                    12,
                    10,
                    22,
                    11
                ],
            },
            'paint': {
                'text-color': '#000000',
                'text-halo-color': '#ffeeee',
                'text-halo-width': 1
            }
        });

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
        })

        // Apply clicked state from URL
        if (clickedId.value !== null) {
            map.setFeatureState(
                { source: 'map-source', id: clickedId.value },
                { clicked: true }
            );
        }

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
            map.setLayoutProperty('nycha-label', 'visibility', visibility);
        });

        // Watch for Subway layer toggle
        watch(showSubway, (newValue) => {
            const visibility = newValue ? 'visible' : 'none';
            map.setLayoutProperty('subway-line', 'visibility', visibility);
            map.setLayoutProperty('subway-label', 'visibility', visibility);
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
                <h2>2025 General Election - Mayor</h2>
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
                <div id="map" class="map">
                    <AddressSearch :map="mapInstance" />
                </div>
                <div class="cuny-logo-wrapper">
                    <a href="https://www.gc.cuny.edu/center-urban-research" target="_blank"><img :src="cunygclogo"
                            alt="CUNY GC Logo" class="cuny-logo"></a>
                </div>
                <Legend :COLOR_SCALE="COLOR_SCALE" :showCityCouncil="showCityCouncil" :showNYCHA="showNYCHA"
                    @update:showCityCouncil="showCityCouncil = $event" @update:showSubway="showSubway = $event"
                    @update:showNYCHA="showNYCHA = $event" />
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

@media (min-width: 600px) {
    .comparison-container {
        flex-direction: row;
    }
}

.details {
    width: 100%;
    height: 65%;
    overflow: auto;
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
    color: #1f2937;
    display: flex;
    flex-direction: column;
    padding: 1rem;
    background-color: white;
}

/* hide h2 and h3 elements on mobile to allow for space*/
.details h2 {
    display: none;
}

.details h3 {
    display: none;
}

@media (min-width: 600px) {
    .details {
        height: 100%;
        width: 25rem;
    }

    .details h2 {
        display: inline;
        margin: 0;
        font-size: 1.4rem;
        line-height: 0.8;
    }

    .details h3 {
        display: inline;
        margin-top: 0;
        font-size: 0.8rem;
        font-weight: 500;
        color: #404040;
    }
}

.map-container {
    position: relative;
    flex: 1;
    order: -1;
}

@media (min-width: 600px) {
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

.filters-section {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    padding-top: 0.4rem;
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
    margin-top: 0.25rem;
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
    /* pointer-events: none; */
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
<script setup lang="ts">
import { ref, watch } from 'vue';
import maplibregl from 'maplibre-gl';

interface Address {
    name: string;
    coords: [number, number];
}

const props = defineProps<{
    map: maplibregl.Map | null;
}>();

const emit = defineEmits<{
    addressSelected: [address: Address];
    coordinatesSelected: [coords: [number, number]];
}>();

const searchQuery = ref('');
const searchResults = ref<Address[]>([]);
const showResults = ref(false);
const isLoading = ref(false);
const isExpanded = ref(false);
const addressMarker = ref<maplibregl.Marker | null>(null);

let searchTimeout: ReturnType<typeof setTimeout> | null = null;

function formatAddress(street: string, borough: string, postalcode: string, housenumber: string): string {
    const parts = [
        housenumber,
        street,
        borough,
        postalcode
    ].filter(Boolean);
    return parts.join(', ');
}

async function searchAddress(keyword: string) {
    if (!keyword.trim()) {
        searchResults.value = [];
        return;
    }

    isLoading.value = true;

    try {
        const url = `https://geosearch.planninglabs.nyc/v2/search?text=${encodeURIComponent(keyword)}&size=5`;
        const response = await fetch(url);
        const data = await response.json();

        searchResults.value = data.features.map((feature: any) => ({
            name: formatAddress(
                feature.properties.street,
                feature.properties.borough,
                feature.properties.postalcode,
                feature.properties.housenumber
            ),
            coords: feature.geometry.coordinates
        }));
    } catch (error) {
        console.error('Search error:', error);
        searchResults.value = [];
    } finally {
        isLoading.value = false;
    }
}

function onInput() {
    if (searchTimeout) {
        clearTimeout(searchTimeout);
    }

    searchTimeout = setTimeout(() => {
        searchAddress(searchQuery.value);
    }, 300);
}

function selectAddress(address: Address) {
    searchQuery.value = address.name;
    showResults.value = false;

    if (props.map) {
        props.map.flyTo({ center: address.coords, zoom: 15 });

        if (addressMarker.value) {
            addressMarker.value.remove();
        }

        addressMarker.value = new maplibregl.Marker({ color: '#ff0000' })
            .setLngLat(address.coords)
            .addTo(props.map);
    }

    emit('addressSelected', address);
}

function handleFocus() {
    showResults.value = true;
    if (searchQuery.value) {
        searchAddress(searchQuery.value);
    }
}

function handleBlur() {
    setTimeout(() => {
        showResults.value = false;
    }, 200);
}

function useGeolocation() {
    if (!navigator.geolocation) {
        alert('Geolocation is not supported by your browser');
        return;
    }

    isLoading.value = true;

    navigator.geolocation.getCurrentPosition(
        (position) => {
            const coords: [number, number] = [position.coords.longitude, position.coords.latitude];

            if (props.map) {
                props.map.flyTo({ center: coords, zoom: 15 });

                if (addressMarker.value) {
                    addressMarker.value.remove();
                }

                addressMarker.value = new maplibregl.Marker({ color: '#ff0000' })
                    .setLngLat(coords)
                    .addTo(props.map);
            }

            emit('coordinatesSelected', coords);
            searchQuery.value = `${coords[1].toFixed(5)}, ${coords[0].toFixed(5)}`;
            isLoading.value = false;
        },
        (_) => {
            alert('Unable to retrieve your location');
            isLoading.value = false;
        }
    );
}

function toggleExpand() {
    isExpanded.value = !isExpanded.value;
}

watch(searchQuery, (newValue) => {
    if (!newValue) {
        searchResults.value = [];
    }
});
</script>

<template>
    <div class="address-search-container">
        <!-- Mobile view: Collapsed by default -->
        <div class="mobile-view">
            <button @click="toggleExpand" class="icon-button search-icon" aria-label="Search address">
                <svg xmlns="http://www.w3.org/2000/svg" class="icon" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd"
                        d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z"
                        clip-rule="evenodd" />
                </svg>
            </button>

            <button @click="useGeolocation" class="icon-button location-icon" :disabled="isLoading"
                aria-label="Use my location">
                <svg xmlns="http://www.w3.org/2000/svg" class="icon" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd"
                        d="M5.05 4.05a7 7 0 119.9 9.9L10 18.9l-4.95-4.95a7 7 0 010-9.9zM10 11a2 2 0 100-4 2 2 0 000 4z"
                        clip-rule="evenodd" />
                </svg>
            </button>
        </div>

        <!-- Desktop view and expanded mobile view -->
        <div class="desktop-view" :class="{ 'expanded': isExpanded }">
            <div class="search-wrapper">
                <svg xmlns="http://www.w3.org/2000/svg" class="search-icon-left" viewBox="0 0 20 20"
                    fill="currentColor">
                    <path fill-rule="evenodd"
                        d="M8 4a4 4 0 100 8 4 4 0 000-8zM2 8a6 6 0 1110.89 3.476l4.817 4.817a1 1 0 01-1.414 1.414l-4.816-4.816A6 6 0 012 8z"
                        clip-rule="evenodd" />
                </svg>

                <input v-model="searchQuery" @input="onInput" @focus="handleFocus" @blur="handleBlur" type="text"
                    placeholder="Search by NYC address" class="search-input" autocomplete="off" />

                <button v-if="isExpanded" @click="toggleExpand" class="close-button" aria-label="Close search">
                    ×
                </button>

                <div v-if="isLoading" class="loading-indicator">
                    <div class="spinner"></div>
                </div>

                <div v-if="showResults && searchResults.length > 0" class="results-dropdown">
                    <div v-for="(result, index) in searchResults" :key="index" @click="selectAddress(result)"
                        class="result-item">
                        {{ result.name }}
                    </div>
                </div>

                <div v-if="showResults && searchQuery && searchResults.length === 0 && !isLoading"
                    class="results-dropdown">
                    <div class="result-item no-results">
                        No addresses found
                    </div>
                </div>
            </div>

            <button @click="useGeolocation" class="location-button" :disabled="isLoading" aria-label="Use my location">
                <svg xmlns="http://www.w3.org/2000/svg" class="icon" viewBox="0 0 20 20" fill="currentColor">
                    <path fill-rule="evenodd"
                        d="M5.05 4.05a7 7 0 119.9 9.9L10 18.9l-4.95-4.95a7 7 0 010-9.9zM10 11a2 2 0 100-4 2 2 0 000 4z"
                        clip-rule="evenodd" />
                </svg>
            </button>
        </div>
    </div>
</template>

<style scoped>
.address-search-container {
    position: absolute;
    top: 10px;
    left: 10px;
    right: 10px;
    z-index: 10;
}

/* Mobile view styles */
.mobile-view {
    display: flex;
    gap: 0.5rem;
}

.desktop-view {
    display: none;
}

@media (min-width: 600px) {
    .mobile-view {
        display: none;
    }

    .desktop-view {
        display: flex;
        gap: 0.5rem;
        width: 35%;
    }
}

/* Expanded mobile view */
@media (max-width: 599px) {
    .desktop-view.expanded {
        display: flex;
        position: fixed;
        top: 10px;
        left: 10px;
        right: 10px;
        z-index: 1000;
    }
}

.icon-button {
    width: 2.4rem;
    height: 2.2rem;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: white;
    border: none;
    border-radius: 4px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    cursor: pointer;
    transition: all 0.2s;
    color: #374151;
}

.icon-button:hover {
    background-color: #f3f4f6;
    color: #2463eb;
}

.icon-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.search-wrapper {
    position: relative;
    flex: 1;
    width: 100%;
}

@media (max-width: 599px) {
    .desktop-view.expanded .search-wrapper {
        min-width: 0;
    }
}

.search-icon-left {
    position: absolute;
    left: 0.625rem;
    top: calc(1.25rem / 2);
    width: 1.25rem;
    height: 1.25rem;
    color: #6b7280;
    pointer-events: none;
}

.search-input {
    width: 100%;
    padding: 0.75rem 0.6rem 0.7rem 2rem;
    background-color: white;
    border: none;
    border-radius: 4px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    font-size: 0.875rem;
    outline: none;
    transition: box-shadow 0.2s;
}

.search-input:focus {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2), 0 0 0 3px rgba(231, 231, 231, 0.1);
}

.close-button {
    position: absolute;
    right: 0.5rem;
    top: 50%;
    transform: translateY(-50%);
    width: 1.5rem;
    height: 1.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    background: none;
    border: none;
    font-size: 1.5rem;
    color: #6b7280;
    cursor: pointer;
    border-radius: 50%;
    transition: all 0.2s;
}

.close-button:hover {
    background-color: #f3f4f6;
    color: #374151;
}

.loading-indicator {
    position: absolute;
    right: 0.75rem;
    top: 50%;
    transform: translateY(-50%);
}

.spinner {
    width: 1rem;
    height: 1rem;
    border: 2px solid #e5e7eb;
    border-top-color: #2463eb;
    border-radius: 50%;
    animation: spin 0.6s linear infinite;
}

@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}

.results-dropdown {
    position: absolute;
    top: calc(100% + 0.25rem);
    left: 0;
    right: 0;
    background: white;
    border-radius: 4px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    max-height: 300px;
    overflow-y: auto;
    z-index: 1000;
}

.result-item {
    padding: 0.75rem;
    cursor: pointer;
    font-size: 0.875rem;
    color: #525252;
    transition: background-color 0.15s;
    border-bottom: 1px solid #f3f4f6;
}

.result-item:last-child {
    border-bottom: none;
}

.result-item:hover {
    background-color: #f9fafb;
}

.result-item.no-results {
    color: #6b7280;
    cursor: default;
    font-style: italic;
}

.result-item.no-results:hover {
    background-color: white;
}

.location-button {
    width: 2.5rem;
    height: 2.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: white;
    border: none;
    border-radius: 4px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    cursor: pointer;
    transition: all 0.2s;
    color: #374151;
    flex-shrink: 0;
}

.location-button:hover {
    background-color: #f3f4f6;
    color: #e11d48;
}

.location-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

.icon {
    width: 1.25rem;
    height: 1.25rem;
}
</style>
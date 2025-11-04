<script setup lang="ts">
import { ref, onMounted } from 'vue'
const { COLOR_SCALE, showCityCouncil, showNYCHA, showSubway } = defineProps(['COLOR_SCALE', 'showCityCouncil', 'showNYCHA', 'showSubway'])

const emit = defineEmits<{
    'update:showCityCouncil': [value: boolean]
    'update:showSubway': [value: boolean]
    'update:showNYCHA': [value: boolean]
}>()

// Detect if screen is mobile/small on mount
const isMobile = ref(false)
const isExpanded = ref(true)

onMounted(() => {
    // Check if viewport is smaller than 768px
    isMobile.value = window.innerWidth < 768
    // Collapse by default on mobile
    if (isMobile.value) {
        isExpanded.value = false
    }
})

function toggleExpanded() {
    isExpanded.value = !isExpanded.value
}

function getLabelText(index: number) {
    const breakpoint = COLOR_SCALE.breakpoints[index];
    const nextBreakpoint = COLOR_SCALE.breakpoints[index + 1];
    if (nextBreakpoint) {
        return `${breakpoint} - ${nextBreakpoint}%`
    }
    return `${breakpoint} - 100`
}
</script>
<template>
    <div class="legend-container">
        <div v-show="isExpanded" class="legend-content">
            <!-- Layer Toggle Section -->
            <div class="layer-toggle">
                <div class="layer-toggle-header">Layers</div>
                <div class="layer-toggle-options">
                    <label class="layer-toggle-option">
                        <input type="checkbox" :checked="showCityCouncil"
                            @change="emit('update:showCityCouncil', ($event.target as HTMLInputElement).checked)" />
                        <span>City Council</span>
                    </label>
                    <label class="layer-toggle-option">
                        <input type="checkbox" :checked="showSubway"
                            @change="emit('update:showSubway', ($event.target as HTMLInputElement).checked)" />
                        <span>Subways</span>
                    </label>
                    <label class="layer-toggle-option">
                        <input type="checkbox" :checked="showNYCHA"
                            @change="emit('update:showNYCHA', ($event.target as HTMLInputElement).checked)" />
                        <span>NYCHA</span>
                    </label>
                </div>
            </div>

            <!-- Legend Section -->
            <div class="legend">
                <!-- Header row with candidate names -->
                <div class="legend-header">
                    <div v-for="candidate in COLOR_SCALE.candidates" :key="`header-${candidate.id}`"
                        class="header-cell">
                        <span class="candidate-name">{{ candidate.label.split(' ')[1] }}</span>
                    </div>
                    <div class="header-cell label-header"></div>
                </div>

                <!-- swatches and labels -->
                <div v-for="(breakpoint, index) in COLOR_SCALE.breakpoints" :key="index"
                    class="legend-row">
                    <div v-for="candidate in COLOR_SCALE.candidates" :key="`${candidate.id}-${index}`"
                        class="swatch-cell">
                        <span class="swatch" :style="`background-color: ${candidate.colors[index + 1]}`"></span>
                    </div>
                    <div class="label-cell">
                        <span>{{ getLabelText(index) }}</span>
                    </div>
                </div>
            </div>
            <p class="subtitle">Blank areas = no votes and/or no population</p>
        </div>
        <button @click="toggleExpanded" class="legend-toggle" aria-label="Toggle legend">
            {{ isExpanded ? 'Hide Legend' : 'Legend' }}
        </button>
    </div>
</template>
<style scoped>
.legend-container {
    position: absolute;
    right: 0.5rem;
    bottom: 0;
    z-index: 3;
    display: flex;
    flex-direction: column;
    align-items: end;
    font-family: monospace;
    pointer-events: none;
}

.legend-toggle {
    pointer-events: auto;
    background-color: rgba(255, 255, 255, 0.9);
    border: 1px solid #ccc;
    border-radius: 4px;
    padding: 0.25rem 0.5rem;
    font-size: 0.75rem;
    font-weight: 600;
    cursor: pointer;
    margin-bottom: 0.25rem;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    transition: background-color 0.2s;
}

.legend-toggle:hover {
    background-color: rgba(255, 255, 255, 1);
}

.legend-content {
    pointer-events: none;
    display: flex;
    flex-direction: column;
    align-items: end;
    gap: 0.5rem;
}

/* Layer Toggle Styles */
.layer-toggle {
    pointer-events: auto;
    background-color: rgba(255, 255, 255, 0.95);
    padding: 0.2rem 0.4rem;
    margin-bottom: 5px;
    border-radius: 4px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    border: 1px solid #ccc;
}

.layer-toggle-header {
    font-size: 0.75rem;
    font-weight: 600;
    color: #333;
    margin-bottom: 0;
}

.layer-toggle-options {
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.layer-toggle-option {
    display: flex;
    align-items: center;
    gap: 0.35rem;
    font-size: 0.75rem;
    color: #333;
    cursor: pointer;
    user-select: none;
}

.layer-toggle-option input[type="checkbox"] {
    cursor: pointer;
    width: 14px;
    height: 14px;
}

.layer-toggle-option:hover {
    color: #0066cc;
}

/* Legend Styles */
.legend {
    display: table;
    border-collapse: collapse;
}

.legend-header {
    display: table-row;
}

.legend-row {
    display: table-row;
}

.header-cell {
    display: table-cell;
    height: 2rem;
    width: 1rem;
    vertical-align: bottom;
    padding: 0;
}

.label-header {
    width: auto;
    padding-left: 0.15rem;
}

.candidate-name {
    display: inline-block;
    font-size: 0.8rem;
    font-weight: 600;
    width: 1rem;
    white-space: nowrap;
    transform: rotate(-50deg) translateY(0.5rem);
    transform-origin: bottom left;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}

.swatch-cell {
    display: table-cell;
    width: 1rem;
    height: 1rem;
    text-align: center;
    vertical-align: middle;
    padding: 0;
}

.swatch {
    display: inline-block;
    width: 1rem;
    height: 1rem;
    vertical-align: middle;
}

.label-cell {
    display: table-cell;
    vertical-align: middle;
    white-space: nowrap;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}

.label-cell span {
    font-size: 0.7rem;
    margin-left: 0.2rem;
    line-height: 1rem;
}

.subtitle {
    font-size: 0.8rem;
    letter-spacing: -1px;
    margin: 0;
    text-align: right;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}
</style>
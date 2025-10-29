<script setup lang="ts">
import { ref, onMounted } from 'vue'
const { COLOR_SCALE } = defineProps(['COLOR_SCALE'])

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
    if (breakpoint === 5) {
        return `${breakpoint} - ${nextBreakpoint}%`
    }
    return `${breakpoint} - ${nextBreakpoint}`
}
</script>
<template>
    <div class="legend-container">
        <button @click="toggleExpanded" class="legend-toggle" aria-label="Toggle legend">
            {{ isExpanded ? 'Hide Legend' : 'Legend' }}
        </button>
        <div v-show="isExpanded" class="legend-content">
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
                <div v-for="(breakpoint, index) in COLOR_SCALE.breakpoints.slice(0, -1)" :key="index" class="legend-row">
                    <div v-for="candidate in COLOR_SCALE.candidates" :key="`${candidate.id}-${index}`"
                        class="swatch-cell">
                        <span class="swatch" :style="`background-color: ${candidate.colors[index]}`"></span>
                    </div>
                    <div class="label-cell">
                        <span>{{ getLabelText(index) }}</span>
                    </div>
                </div>
            </div>
            <p class="subtitle">Blank areas = no votes and/or no population</p>
        </div>
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
}

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
    margin: 0.15rem 0 0 0;
    text-align: right;
    text-shadow: 1px 1px 2px rgba(255, 255, 255, 0.8);
}
</style>
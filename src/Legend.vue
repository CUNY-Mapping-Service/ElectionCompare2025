<script setup lang="ts">
import { ref } from 'vue'
const { COLOR_SCALE } = defineProps(['COLOR_SCALE'])
const isExpanded = ref(true)

function getLabelText(index: number) {
    const breakpoint = COLOR_SCALE.breakpoints[index];
    const nextBreakpoint = COLOR_SCALE.breakpoints[index + 1];
    return nextBreakpoint ? `${breakpoint} – ${nextBreakpoint}` : `${breakpoint}%`;
}
</script>
<template>
    <div class="legend-container">
        <div class="legend">
            <!-- Header row with candidate names -->
            <div class="legend-header">
                <div v-for="candidate in COLOR_SCALE.candidates" :key="`header-${candidate.id}`" class="header-cell">
                    <span class="candidate-name">{{ candidate.label.split(' ')[1] }}</span>
                </div>
                <div class="header-cell label-header"></div>
            </div>
            
            <!-- swatches and labels -->
            <div v-for="(breakpoint, index) in COLOR_SCALE.breakpoints" :key="index" class="legend-row">
                <div v-for="candidate in COLOR_SCALE.candidates" :key="`${candidate.id}-${index}`" class="swatch-cell">
                    <span class="swatch" :style="`background-color: ${candidate.colors[index]}`"></span>
                </div>
                <div class="label-cell">
                    <span>{{ getLabelText(index) }}</span>
                </div>
            </div>
        </div>
        <p class="subtitle">Blank areas = no votes and/or no population</p>
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
}

.label-cell span {
    font-size: 0.7rem;
    margin-left: 0.2rem;
    line-height: 1rem;
}

.subtitle {
    font-size: 0.8rem;
    letter-spacing: -0.5px;
    margin: 0.15rem 0 0 0;
    text-align: right;
}
</style>
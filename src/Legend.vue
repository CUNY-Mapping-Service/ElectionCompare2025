<script setup lang="ts">
import { ref } from 'vue'

const { COLOR_SCALE } = defineProps(['COLOR_SCALE'])
const isExpanded = ref(true)

// Generate legend items from breakpoints and colors for a given candidate
function getLegendItems(colors: string[]) {
    return COLOR_SCALE.breakpoints.map((breakpoint: number, index: number) => {
        const nextBreakpoint = COLOR_SCALE.breakpoints[index + 1];
        const label = nextBreakpoint
            ? `${breakpoint} – ${nextBreakpoint}`
            : `${breakpoint}%`;

        return {
            label,
            color: colors[index]
        };
    });
}
</script>

<template>
    <div class="legend-container">
        <!-- <button class="toggle-btn" @click="isExpanded = !isExpanded" :aria-expanded="isExpanded">
            <span class="toggle-icon">{{ isExpanded ? '▼' : '▶' }}</span>
            <span class="toggle-label">Legend</span>
        </button> -->

        <div class="legend">
            <div v-for="candidate in COLOR_SCALE.candidates" :key="candidate.id" class="candidate">
                <h3 class="candidate-label">{{ candidate.label.split(' ')[1] }}</h3>
                <div v-for="(item, index) in getLegendItems(candidate.colors)" :key="index">
                    <span class="swatch" :style="`background-color: ${item.color}`"></span>
                </div>
            </div>
            <div class="candidate" style="width: auto;">
                <h3 class="candidate-label">‎</h3>
                <div class="breakpoint" v-for="(item, index) in getLegendItems(COLOR_SCALE.candidates[0].colors)"
                    :key="index">
                    <p>{{ item.label }}</p>
                </div>
            </div>
        </div>

        <p class="subtitle">Blank areas = no votes and/or no population</p>
    </div>
</template>

<style scoped>
.legend-container {
    display: flex;
    flex-direction: column;
}

.toggle-btn {
    align-self: flex-start;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    background: none;
    border: none;
    padding: 0.5rem 0;
    cursor: pointer;
    font-size: 1rem;
    font-weight: 600;
    color: inherit;
    transition: opacity 0.2s;
}

.toggle-btn:hover {
    opacity: 0.7;
}

.toggle-icon {
    display: inline-block;
    width: 1rem;
    transition: transform 0.2s;
}

.legend {
    display: flex;
    gap: 2px;
    margin-top: 2rem;
}

.candidate {
    width: 1.1rem;
}

.candidate-label {
    margin: 0;
    font-size: 1rem;
    font-weight: 600;
    transform: rotate(-50deg);
}

.swatch {
    display: block;
    width: 1rem;
    height: 1rem;
    flex-shrink: 0;
}

.breakpoint {
    display: flex;
    flex-direction: row;
    text-wrap: wrap;
    margin-top: 0rem;
    gap: 0.3rem;
}

.breakpoint p {
    margin-top: 0;
    line-height: 1.1;
    font-size: 0.9rem;
}

.subtitle {
    font-size: 0.8rem;
    letter-spacing: -0.5px;
    margin-top: 0.5rem;
}
</style>
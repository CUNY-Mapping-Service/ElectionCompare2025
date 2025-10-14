<!-- eslint-disable @typescript-eslint/no-explicit-any -->
<script setup lang="ts">
import { computed } from 'vue';

interface FeatureProperties {
    [key: string]: any;
}

interface YearSettings {
    year: number;
    candidate: string;
    opponent: string;
    candidateKey: string;
    opponentKey: string;
    totalKey: string;
    label: string;
    opponentLabel: string;
}

interface Props {
    hoveredData: {
        id: string | number;
        properties: FeatureProperties;
    } | null;
    beforeSettings: YearSettings;
    afterSettings: YearSettings;
    idKey: string;
}

const props = defineProps<Props>();

const edLabel = computed(() => {
    const id = props.hoveredData?.properties[props.idKey]
    if (id) {
        const ad = String(id).slice(0, 2)
        const ed = String(id).slice(2)
        return `ED ${ed} in AD ${ad}`
    }
    return ''
})

// Computed values for percentages
const candidateBefore = computed(() => {
    if (!props.hoveredData) return 0;
    const pct = (props.hoveredData.properties[props.beforeSettings.candidateKey] /
        props.hoveredData.properties[props.beforeSettings.totalKey]) * 100;
    return isNaN(pct) ? 0 : pct;
});

const opponentBefore = computed(() => {
    if (!props.hoveredData) return 0;
    const pct = (props.hoveredData.properties[props.beforeSettings.opponentKey] /
        props.hoveredData.properties[props.beforeSettings.totalKey]) * 100;
    return isNaN(pct) ? 0 : pct;
});

const candidateAfter = computed(() => {
    if (!props.hoveredData) return 0;
    const pct = (props.hoveredData.properties[props.afterSettings.candidateKey] /
        props.hoveredData.properties[props.afterSettings.totalKey]) * 100;
    return isNaN(pct) ? 0 : pct;
});

const opponentAfter = computed(() => {
    if (!props.hoveredData) return 0;
    const pct = (props.hoveredData.properties[props.afterSettings.opponentKey] /
        props.hoveredData.properties[props.afterSettings.totalKey]) * 100;
    return isNaN(pct) ? 0 : pct;
});

// Computed values for deltas
const candidatePctDelta = computed(() => candidateAfter.value - candidateBefore.value);
const opponentPctDelta = computed(() => opponentAfter.value - opponentBefore.value);

const candidateNumDelta = computed(() => {
    if (!props.hoveredData) return 0;
    return props.hoveredData.properties[props.afterSettings.candidateKey] -
        props.hoveredData.properties[props.beforeSettings.candidateKey];
});

const opponentNumDelta = computed(() => {
    if (!props.hoveredData) return 0;
    return props.hoveredData.properties[props.afterSettings.opponentKey] -
        props.hoveredData.properties[props.beforeSettings.opponentKey];
});

const totalDelta = computed(() => {
    if (!props.hoveredData) return 0;
    return props.hoveredData.properties[props.afterSettings.totalKey] -
        props.hoveredData.properties[props.beforeSettings.totalKey];
});

// Helper function to format percentage
const formatPct = (value: number) => {
    return value.toFixed(1) + '%';
};

// Helper function to format delta with sign
const formatDelta = (value: number, suffix: string = '') => {
    const sign = value >= 0 ? '+' : '';
    return sign + value.toFixed(1) + suffix;
};

// Helper function to format number delta with sign
const formatNumDelta = (value: number) => {
    const sign = value >= 0 ? '+' : '';
    return sign + value.toLocaleString();
};

// Helper function to get color class based on value
const getDeltaClass = (value: number) => {
    if (value > 0) return 'positive';
    if (value < 0) return 'negative';
    return '';
};

// Show/hide container based on hover
const isVisible = computed(() => props.hoveredData !== null);
</script>

<template>
    <div class="infobox-container" v-if="isVisible">
        <div class="table-wrapper">
            <table class="table table-bordered">
                <thead>
                    <tr class="table-header">
                        <th colspan="3">
                            {{ beforeSettings.year }}/{{ afterSettings.year }} Vote Share
                            <span class="ed">{{ edLabel }}</span>
                        </th>
                    </tr>
                    <tr>
                        <th>{{ beforeSettings.year }} vote results</th>
                        <th>{{ afterSettings.year }} vote results</th>
                        <th>% Point Change</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td class="pct-value">{{ beforeSettings.candidate }} {{ formatPct(candidateBefore) }}</td>
                        <td class="pct-value">{{ afterSettings.candidate }} {{ formatPct(candidateAfter) }}</td>
                        <td class="pct-value" :class="getDeltaClass(candidatePctDelta)">
                            {{ afterSettings.label }} {{ formatDelta(candidatePctDelta, ' pts') }}
                        </td>
                    </tr>
                    <tr>
                        <td class="pct-value">{{ beforeSettings.opponent }} {{ formatPct(opponentBefore) }}</td>
                        <td class="pct-value">{{ afterSettings.opponent }} {{ formatPct(opponentAfter) }}</td>
                        <td class="pct-value" :class="getDeltaClass(opponentPctDelta)">
                            {{ afterSettings.opponentLabel }} {{ formatDelta(opponentPctDelta, ' pts') }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>

        <div class="table-wrapper">
            <table class="table table-bordered">
                <thead>
                    <tr>
                        <th>{{ beforeSettings.year }} vote results</th>
                        <th>{{ afterSettings.year }} vote results</th>
                        <th>Numeric Change</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td class="num-value">{{ beforeSettings.candidate }} {{
                            hoveredData?.properties[beforeSettings.candidateKey]?.toLocaleString() || 0 }}</td>
                        <td class="num-value">{{ afterSettings.candidate }} {{
                            hoveredData?.properties[afterSettings.candidateKey]?.toLocaleString() || 0 }}</td>
                        <td class="num-value" :class="getDeltaClass(candidateNumDelta)">
                            {{ afterSettings.label }} {{ formatNumDelta(candidateNumDelta) }}
                        </td>
                    </tr>
                    <tr>
                        <td class="num-value">{{ beforeSettings.opponent }} {{
                            hoveredData?.properties[beforeSettings.opponentKey]?.toLocaleString() || 0 }}</td>
                        <td class="num-value">{{ afterSettings.opponent }} {{
                            hoveredData?.properties[afterSettings.opponentKey]?.toLocaleString() || 0 }}</td>
                        <td class="num-value" :class="getDeltaClass(opponentNumDelta)">
                            {{ afterSettings.opponentLabel }} {{ formatNumDelta(opponentNumDelta) }}
                        </td>
                    </tr>
                    <tr style="font-weight: bold;">
                        <td class="num-value">Total {{
                            hoveredData?.properties[beforeSettings.totalKey]?.toLocaleString() ||
                            0 }}</td>
                        <td class="num-value">Total {{ hoveredData?.properties[afterSettings.totalKey]?.toLocaleString()
                            ||
                            0 }}</td>
                        <td class="num-value">
                            Total {{ formatNumDelta(totalDelta) }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </div>
</template>

<style scoped>
.infobox-container {
    position: absolute;
    left: 5px;
    bottom: 5px;
    z-index: 99;
    width: min(30rem, calc(100% - 3rem));
    background-color: white;
    box-shadow: 2px 2px 8px darkgray;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 5px;
}

.table-wrapper {
    background-color: white;
}

@media screen and (max-width: 990px) {
    .infobox-container {
        font-size: small;
    }
}

.table {
    width: 100%;
    margin-bottom: 0;
}

.table-bordered {
    border: 1px solid #dee2e6;
}

.table-header th {
    background-color: lightgray;
    padding: 5px !important;
    text-align: left;
}

.table thead th {
    border-bottom: 2px solid #dee2e6;
    padding: 5px;
}

.table tbody td {
    border-top: 1px solid #dee2e6;
}

td,
th {
    padding: 6px 8px !important;
    font-size: 1em;
}

.positive {
    background-color: #c2ffc7;
    font-weight: 600;
}

.negative {
    background-color: #faafaf;
    font-weight: 600;
}

.ed {
    font-weight: normal;
}
</style>